---
name: questdb-tsbs-benchmark
description: Run full TSBS (Time Series Benchmark Suite) benchmarks against QuestDB in Docker over either ingestion protocol, QWP (binary) or ILP (line protocol text), including prerequisite checks, TSBS build, data generation, loading, query generation, and benchmark execution. Use when asked to set up or run QuestDB TSBS performance tests end-to-end.
---

# TSBS Benchmark for QuestDB

Run the full TSBS (Time Series Benchmark Suite) against QuestDB running in Docker.
Handle all prerequisites, data generation, loading, and query benchmarking.

QuestDB accepts data over three ingestion transports, and TSBS benchmarks all
of them:

| | **QWP** | **ILP over HTTP** | **ILP over TCP** |
|---|---|---|---|
| Shape | binary, columnar | line protocol text | line protocol text |
| Transport | WebSocket, port 9000 | HTTP, port 9000 | TCP, port 9009 |
| Generator format | `questdb-qwp` | `questdb` | `questdb` |
| Loader flag | `--protocol=qwp` (default) | `--protocol=ilp-http` | `--protocol=ilp` |
| Delivery | acknowledged by the server | server processed the batch | fire and forget |

QWP's reported row count is what the server confirmed, not just what was written
to a socket. But whether it is "faster" than ILP depends entirely on topology,
and this skill runs the loader and the server on the same box, which is the case
that understates QWP the most (see next).

### Co-located ingestion understates QWP

This skill benchmarks with the loader and QuestDB on one instance, matching the
published comparison posts. On that setup QWP and a well-configured ILP/TCP come
out roughly level on rows sent, and it is a measurement artefact, not a property
of the protocols.

QWP is a binary columnar protocol: the client encodes every row into the wire
format before sending. ILP just writes text, which is already the wire format,
so its client does almost no work. When client and server share the CPU, QWP's
encoding competes with the server for cores and the two roughly cancel. Measured
on a 32 vCPU box, the QWP loader used ~10 cores, the ILP loader ~1.7.

Put the client on its own instance and the picture separates: over a real
network both ILP transports saturate the link at ~5.3M rows/s while QWP sustains
9-12M, because line protocol text is ~3.4x larger on the wire. QWP is faster
over a network, level on a shared box.

So report a co-located QWP number as exactly that, and do not present it as
QWP's ceiling. `--qwp-preencode-replay` is a diagnostic that removes the client
encoding from the timed interval (it pre-builds the frames, then replays them);
on the same hardware it took QWP from ~9M to ~18M over a network and ~48M on
localhost, which is the server's true ingest capacity.

### Pin the CPUs when co-located, and keep the server's budget fixed

When the loader and QuestDB share a box they fight for the same cores, and that
fight is not symmetric: the QWP client encodes every row while the ILP client
does almost none, so the result flatters whichever protocol leaves more cores
for the server. The loaders are light (ILP used ~1.7 cores, a replay send is
lighter), so give the client a couple of cores and the server the rest, on
disjoint sets so both protocols see the same, non-competing budget:

- Start QuestDB on almost all the cores and size its pools to match:
  `docker run --cpuset-cpus=0-29 -e QDB_SHARED_WORKER_COUNT=29 -e QDB_LINE_TCP_IO_WORKER_COUNT=29 -e QDB_LINE_TCP_WRITER_WORKER_COUNT=29 ...`
- Run the loader on the remaining cores: `taskset -c 30-31 tsbs_load_questdb ...`.

Keep the server's core count identical whether the client is co-located or on
its own box: give it the same 30 cores over the network too. If the co-located
server gets 30 cores but the networked server gets all 32, the network run is
measuring a bigger server, not the network, and the two topologies stop being
comparable.

### Sending tags as VARCHAR at very high cardinality

`--qwp-tags-as-varchar` sends tag columns as VARCHAR strings instead of QWP
SYMBOLs, so no per-frame symbol dictionary is shipped. The server still stores
the columns as SYMBOL as long as the table already exists with SYMBOL columns
(pre-create it, or let an earlier SYMBOL load create it). At low cardinality
this makes each row larger on the wire; at very high cardinality it avoids the
per-frame dictionary growth that otherwise inflates QWP frames. The table
definition is unchanged, so queries and storage are unaffected.

**An ILP/TCP number depends on the server's thread pools, so record them.**
Measured on a 32 vCPU r8a.8xlarge, 69.1M rows, 32 workers, send rates:

| server | ILP/TCP | ILP/HTTP |
|---|---|---|
| QuestDB 9.4.3 release, defaults | 9.3-12.5M rows/s | 6.8-6.9M rows/s |
| 9.4.4-SNAPSHOT nightly, defaults | 1.7M rows/s | 6.9-7.0M rows/s |
| the same nightly, `QDB_LINE_TCP_IO_WORKER_COUNT=16` | 8.0M rows/s | - |

ILP/TCP varies sevenfold across builds and settings while ILP/HTTP does not
move. The nightly gives the ILP/TCP pools 2 threads where the shared pools each
get 31; the release build has no separate ILP pool and serves TCP from the
shared ones. Before reporting an ILP/TCP figure, check the pool sizes:

```bash
QPID=$(pgrep -f questdb | head -1)
ps -L -o comm= -p "$QPID" | sed 's/_[0-9]*$//' | sort | uniq -c | sort -rn | head -8
```

If `ilpio` is far smaller than `shared-network`, say so with the number, or use
`ilp-http`, which the shared pools serve and which measured 6.8-7.0M rows/s on
both builds with no tuning.

Also note that ILP/TCP's send rate flatters it most, being fire-and-forget: on
the release build it sent 9.3-12.5M rows/s but committed only 3.2-3.9M, where
HTTP sent 6.8-6.9M and committed 5.0-5.1M.

## Step 0: Choose The Protocols

Ask the operator which ingestion protocol to benchmark: `qwp`, `ilp-http`,
`ilp`, or `all`. When running unattended, default to `qwp`. Everything below
branches on this choice, so settle it before generating any data: QWP wants the
binary format and the ILP transports want text, and at scale 4000 those are
3.6 GB and 12 GB on disk respectively.

```bash
PROTOCOL=qwp    # qwp | ilp-http | ilp | all
```

For a protocol comparison, `all` is the useful setting: it loads the same rows
over each transport in turn and reports them side by side. Both ILP transports
read the same text file, so `all` needs only the two data files.

Queries have their own transport, independent of the ingestion one, chosen in
Step 7. Data written over either protocol can be queried over any of them.

## Prerequisites Check And Install

### 1. Docker
Check if Docker is installed and the daemon is running:
```bash
docker --version && docker info --format '{{.ServerVersion}}'
```
If Docker is not installed, install it (Ubuntu/Debian):
```bash
sudo apt-get update -qq
sudo apt-get install -y -qq ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -qq
sudo apt-get install -y -qq docker-ce docker-ce-cli containerd.io docker-buildx-plugin
```

### 2. Go (needed to build TSBS)
TSBS requires **Go 1.23 or newer**. Check what is installed:
```bash
go version
```
If Go is missing or older than 1.23, install the current stable release. This
picks the right architecture, so it works on both x86 and Graviton instances:
```bash
GO_VERSION=$(curl -fsSL "https://go.dev/VERSION?m=text" | head -1)
GO_ARCH=$(dpkg --print-architecture)
curl -fsSL "https://go.dev/dl/${GO_VERSION}.linux-${GO_ARCH}.tar.gz" -o /tmp/go.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf /tmp/go.tar.gz && rm /tmp/go.tar.gz
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
go version
```

### 3. Build tools
```bash
sudo apt-get install -y -qq make gcc gzip
```

## Step 1: Start QuestDB In Docker

QWP support ships in the nightly image. Use it for every run so both protocols
are measured against the same server:
```bash
QUESTDB_IMAGE=questdb/questdb:nightly
```
Stop and remove any existing QuestDB container first to start clean:
```bash
sudo docker rm -f questdb 2>/dev/null || true
```
Pull the image and start it:
```bash
sudo docker pull $QUESTDB_IMAGE
sudo docker run -d --name questdb \
  -p 9000:9000 -p 9009:9009 -p 8812:8812 -p 9003:9003 \
  $QUESTDB_IMAGE
```
Verify it is running and answering:
```bash
sudo docker ps --filter name=questdb --format '{{.Status}}'
curl -s --retry 30 --retry-delay 1 --retry-all-errors -o /dev/null -w "ping:%{http_code}\n" http://127.0.0.1:9000/ping
```
`ping:204` means QuestDB is up.

## Step 2: Clone And Build TSBS

QWP support is on the `jv/adding_qwp` branch until it merges to master. Once it
has merged, drop the `--branch` argument.
```bash
cd /home/ubuntu
git clone --branch jv/adding_qwp https://github.com/questdb/tsbs.git
cd /home/ubuntu/tsbs
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
make tsbs_generate_data tsbs_generate_queries tsbs_load_questdb tsbs_run_queries_questdb
```
Build only the four binaries needed for QuestDB benchmarking into `bin/`.

## Step 3: Confirm The Server Speaks QWP

Skip this when `PROTOCOL=ilp`. It takes seconds and prevents discovering a
protocol mismatch after generating gigabytes of data. Generate ten rows, load
them, and check they landed:
```bash
TSBS_BIN=/home/ubuntu/tsbs/bin

$TSBS_BIN/tsbs_generate_data --use-case=cpu-only --seed=123 --scale=1 \
  --timestamp-start="2016-01-01T00:00:00Z" --timestamp-end="2016-01-01T00:01:40Z" \
  --log-interval=10s --format=questdb-qwp > /tmp/qwp-smoke.qwp

$TSBS_BIN/tsbs_load_questdb --file=/tmp/qwp-smoke.qwp --workers=1

# WAL apply is asynchronous, so poll rather than reading the count once
for i in $(seq 1 30); do
  N=$(curl -s -G --data-urlencode "query=select count from cpu" http://127.0.0.1:9000/exec | grep -o '\[\[[0-9]*' | tr -d '[')
  if [ "$N" = "10" ]; then break; fi
  sleep 1
done
echo "smoke rows: $N"
curl -s -G --data-urlencode "query=drop table if exists cpu" http://127.0.0.1:9000/exec
rm -f /tmp/qwp-smoke.qwp
```
A count of 10 means QWP works. A count of `0` read immediately after the load
is not a failure, it means the count was taken before the WAL was applied,
which is why the loop polls. A `PARSE_ERROR` such as `invalid column type
code` means the image is too old for the client's QWP version: stop and report
that rather than falling back silently, since an ILP number reported as a QWP
number is worse than no number.

## Step 4: Generate Benchmark Data

IMPORTANT: Do not compress (`gzip`) the data file. Compression/decompression adds CPU overhead that skews benchmark results, especially on smaller machines.

For QWP, generate the binary `questdb-qwp` format. The loader sends it without
parsing text, which is what a QWP benchmark should measure:
```bash
/home/ubuntu/tsbs/bin/tsbs_generate_data \
  --use-case="cpu-only" \
  --seed=123 \
  --scale=4000 \
  --timestamp-start="2016-01-01T00:00:00Z" \
  --timestamp-end="2016-01-02T00:00:00Z" \
  --log-interval="10s" \
  --format="questdb-qwp" \
  > /tmp/questdb-data.qwp
```
For ILP, generate the line protocol text format:
```bash
/home/ubuntu/tsbs/bin/tsbs_generate_data \
  --use-case="cpu-only" \
  --seed=123 \
  --scale=4000 \
  --timestamp-start="2016-01-01T00:00:00Z" \
  --timestamp-end="2016-01-02T00:00:00Z" \
  --log-interval="10s" \
  --format="questdb" \
  > /tmp/questdb-data.txt
```
Both describe the same 34.5M rows and 345.6M metrics: about 3.6 GB as binary,
about 12 GB as text. For `PROTOCOL=both`, generate both files and make sure the
instance has room for roughly 16 GB of data plus the database itself.

## Step 5: Load Data Into QuestDB

Use as many workers as CPU cores available (up to 32). Capture the load summary and report ingestion throughput in rows/s (and metrics/s):
```bash
WORKERS=$(nproc)
if [ "$WORKERS" -gt 32 ]; then WORKERS=32; fi
```
QWP. The loader recognises the binary file by its header, so no format flag is
needed, and `qwp` is the default protocol:
```bash
/home/ubuntu/tsbs/bin/tsbs_load_questdb \
  --file=/tmp/questdb-data.qwp \
  --workers=$WORKERS \
  2>&1 | tee /tmp/questdb-load-qwp.log
```
ILP over HTTP, the ILP baseline:
```bash
/home/ubuntu/tsbs/bin/tsbs_load_questdb \
  --file=/tmp/questdb-data.txt \
  --protocol=ilp-http \
  --workers=$WORKERS \
  2>&1 | tee /tmp/questdb-load-ilp-http.log
```
ILP over TCP, the legacy transport:
```bash
/home/ubuntu/tsbs/bin/tsbs_load_questdb \
  --file=/tmp/questdb-data.txt \
  --protocol=ilp \
  --workers=$WORKERS \
  2>&1 | tee /tmp/questdb-load-ilp.log
```
Report the mean rates from the summary, `overall row/s` for rows/second and
`overall metric/s` for metrics/second:
```bash
grep -E "loaded .*metrics|loaded .*rows" /tmp/questdb-load-*.log
```
Then confirm what the server actually committed, which is the number worth
quoting. QuestDB applies the write-ahead log asynchronously, so a count taken
the instant a load finishes reads low, and an `ilp` run returns as soon as the
bytes are written. Poll until the count reaches the expected total:
```bash
EXPECTED=34560000
for i in $(seq 1 600); do
  N=$(curl -s -G --data-urlencode "query=select count from cpu" http://127.0.0.1:9000/exec | grep -o '\[\[[0-9]*' | tr -d '[')
  echo "committed: $N"
  if [ "$N" -ge "$EXPECTED" ]; then break; fi
  sleep 1
done
```
Time from the start of the load to the moment that count is reached: that is the
committed-rows throughput, and the only figure comparable across all three
transports.

When benchmarking `both`, drop the table between the two loads so each starts
from empty, and load ILP first so the QWP run is not the one paying for a cold
server:
```bash
curl -s -G --data-urlencode "query=drop table if exists cpu" http://127.0.0.1:9000/exec
```

## Step 6: Generate Query Files

Query generation is protocol-independent: always use `--format=questdb`.
Generate 1000 queries for all 16 cpu-only query types. Keep all query files uncompressed:
```bash
TSBS_BIN=/home/ubuntu/tsbs/bin
COMMON_ARGS="--use-case=cpu-only --seed=123 --scale=4000 --timestamp-start=2016-01-01T00:00:00Z --timestamp-end=2016-01-02T00:00:01Z --queries=1000 --format=questdb"

for QTYPE in \
  cpu-max-all-1 cpu-max-all-8 cpu-max-all-32-24 \
  single-groupby-1-1-1 single-groupby-1-1-12 single-groupby-1-8-1 \
  single-groupby-5-1-1 single-groupby-5-1-12 single-groupby-5-8-1 \
  double-groupby-1 double-groupby-5 double-groupby-all \
  high-cpu-1 high-cpu-all \
  lastpoint groupby-orderby-limit; do
  $TSBS_BIN/tsbs_generate_queries $COMMON_ARGS --query-type="$QTYPE" > "/tmp/questdb-queries-${QTYPE}.txt"
done
```

## Step 7: Run Query Benchmarks

IMPORTANT: Use one worker for queries. QuestDB parallelizes queries internally (multi-threaded execution), so multiple client workers over-subscribe CPU and produce misleading results.

Queries can go over three transports, chosen with `--query-protocol`:

| Value | Transport | Port |
|---|---|---|
| `pg` | PostgreSQL wire (default) | 8812 |
| `http` | REST `/exec`, JSON results | 9000 |
| `qwp` | QuestDB Wire Protocol, columnar batches | 9000 |

All three run the same SQL with the same bind parameters, and the query files are protocol-independent, so the same files feed every transport and the run can be repeated per transport to compare them. Ask the operator which to use, or default to `pg`, which is what QuestDB's published TSBS numbers use. Set it once:
```bash
QUERY_PROTOCOL=pg    # pg | http | qwp
```

```bash
TSBS_BIN=/home/ubuntu/tsbs/bin

for QTYPE in \
  cpu-max-all-1 cpu-max-all-8 cpu-max-all-32-24 \
  single-groupby-1-1-1 single-groupby-1-1-12 single-groupby-1-8-1 \
  single-groupby-5-1-1 single-groupby-5-1-12 single-groupby-5-8-1 \
  double-groupby-1 double-groupby-5 double-groupby-all \
  high-cpu-1 high-cpu-all \
  lastpoint groupby-orderby-limit; do
  echo "=== $QTYPE ==="
  $TSBS_BIN/tsbs_run_queries_questdb \
    --file="/tmp/questdb-queries-${QTYPE}.txt" \
    --workers=1 \
    --print-interval=0 \
    --query-protocol=$QUERY_PROTOCOL
  echo ""
done
```
The data on disk is identical whichever ingestion protocol wrote it, so the query suite needs running only once per query transport being compared, not once per ingestion protocol.

## Cleanup

When done benchmarking:
```bash
sudo docker rm -f questdb
rm -f /tmp/questdb-data.txt /tmp/questdb-data.qwp /tmp/questdb-queries-*.txt /tmp/questdb-load-*.log
```

## Reporting Results

State which protocol produced each ingestion number. Never present an ILP
figure as a QWP figure or the reverse. For a `both` run, report the committed
rows/s for each and note that the ILP loader's own summary overstates its rate,
because it counts bytes handed to a socket rather than rows the server
confirmed.

## Notes

- `cpu-only` with `scale=4000` and a 1-day window is a solid general benchmark.
- For `--use-case`, also try `devops` or `iot` (each has different query types).
- Run `tsbs_generate_queries --help` to see the full matrix of use-case + query-type combinations.
- Port `9000` is Web Console, QWP ingestion and QWP/HTTP queries; `9009` is ILP ingestion; `8812` is PostgreSQL wire queries.
- Ingestion uses `9000` (QWP) or `9009` (ILP); queries use `8812` (`pg`) or `9000` (`http`, `qwp`).
- Ingestion protocol and query protocol are independent: data written over ILP can be queried over QWP and the reverse.
- QWP keeps scaling as workers are added, where ILP flattens out once the server's text parsing saturates, so do not tune the worker count on an ILP run and reuse it for QWP.
- Useful extra loader flags: `--qwp-await-ack` waits for the server to acknowledge every batch before counting it, `--qwp-addr` points at a different host or a comma-separated list for failover, and `--qwp-sf-dir` turns on durable store-and-forward, which trades throughput for durability and should be reported as a separate mode.
