---
name: questdb-tsbs-benchmark
description: Run full TSBS (Time Series Benchmark Suite) benchmarks against QuestDB in Docker over either ingestion protocol, QWP (binary) or ILP (line protocol text), including prerequisite checks, TSBS build, data generation, loading, query generation, and benchmark execution. Use when asked to set up or run QuestDB TSBS performance tests end-to-end.
---

# TSBS Benchmark for QuestDB

Run the full TSBS (Time Series Benchmark Suite) against QuestDB running in Docker.
This skill handles all prerequisites, data generation, loading, and query benchmarking.

QuestDB accepts data over two ingestion protocols, and TSBS benchmarks both:

| | **QWP** | **ILP** |
|---|---|---|
| Shape | binary, columnar | line protocol text |
| Transport | WebSocket, port 9000 | TCP, port 9009 |
| Generator format | `questdb-qwp` | `questdb` |
| Loader flag | `--protocol=qwp` (default) | `--protocol=ilp` |
| Delivery | acknowledged by the server | fire and forget |

Use QWP unless there is a reason not to: it is the faster path, and its
reported row count is what the server confirmed rather than what was written
to a socket. Keep ILP for a baseline comparable with other TSBS targets.

## Step 0: Choose the protocols

Ask the operator which ingestion protocol to benchmark: `qwp`, `ilp`, or `both`.
If running unattended, default to `qwp`. Everything below branches on this
choice, so settle it before generating any data - the two protocols need
different data files, and at scale 4000 that is 12 GB versus 3.6 GB on disk.

```bash
PROTOCOL=qwp    # qwp | ilp | both
```

Queries have their own transport, independent of the ingestion one, chosen in
Step 7. Data written over either protocol can be queried over any of them.

## Prerequisites check and install

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

## Step 1: Start QuestDB in Docker

QWP support ships in the nightly image. Use it for every run so that both
protocols are measured against the same server:
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

## Step 2: Clone and build TSBS

QWP support is on the `jv/adding_qwp` branch until it merges to master. Once
it has merged, drop the `--branch` argument.
```bash
cd /home/ubuntu
git clone --branch jv/adding_qwp https://github.com/questdb/tsbs.git
cd /home/ubuntu/tsbs
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
make tsbs_generate_data tsbs_generate_queries tsbs_load_questdb tsbs_run_queries_questdb
```
This builds only the 4 binaries needed for QuestDB benchmarking into `bin/`.

## Step 3: Confirm the server speaks QWP

Skip this when `PROTOCOL=ilp`. It takes seconds and prevents discovering a
protocol mismatch after generating gigabytes of data. Generate ten rows,
load them, and check they landed:
```bash
TSBS_BIN=/home/ubuntu/tsbs/bin

$TSBS_BIN/tsbs_generate_data --use-case=cpu-only --seed=123 --scale=1 \
  --timestamp-start="2016-01-01T00:00:00Z" --timestamp-end="2016-01-01T00:01:40Z" \
  --log-interval=10s --format=questdb-qwp > /tmp/qwp-smoke.qwp

$TSBS_BIN/tsbs_load_questdb --file=/tmp/qwp-smoke.qwp --workers=1

curl -s -G --data-urlencode "query=select count from cpu" http://127.0.0.1:9000/exec
curl -s -G --data-urlencode "query=drop table if exists cpu" http://127.0.0.1:9000/exec
rm -f /tmp/qwp-smoke.qwp
```
A count of 10 means QWP works. A `PARSE_ERROR` such as `invalid column type
code` means the image is too old for the client's QWP version: stop and report
that rather than falling back silently, since an ILP number reported as a QWP
number is worse than no number.

## Step 4: Generate benchmark data

IMPORTANT: Do NOT compress (gzip) the data file. Compression/decompression adds CPU
overhead that skews benchmark results, especially on smaller machines.

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

## Step 5: Load data into QuestDB

Use as many workers as CPU cores available (up to 32). Detect with `nproc`:
```bash
WORKERS=$(nproc)
if [ "$WORKERS" -gt 32 ]; then WORKERS=32; fi
```
QWP. The loader recognises the binary file by its header, so no format flag is
needed, and `qwp` is the default protocol:
```bash
/home/ubuntu/tsbs/bin/tsbs_load_questdb \
  --file=/tmp/questdb-data.qwp \
  --workers=$WORKERS
```
ILP:
```bash
/home/ubuntu/tsbs/bin/tsbs_load_questdb \
  --file=/tmp/questdb-data.txt \
  --protocol=ilp \
  --workers=$WORKERS
```
Report the `overall row/s` and `overall metric/s` figures from the summary.

Then confirm what the server actually committed, which is the number worth
quoting:
```bash
curl -s -G --data-urlencode "query=select count from cpu" http://127.0.0.1:9000/exec
```
It should read 34560000. Poll it until it stops rising: an ILP run in
particular returns as soon as the bytes are written, so the server can still be
applying rows after the loader has exited.

When benchmarking `both`, drop the table between the two loads so each starts
from empty, and load ILP first so the QWP run is not the one paying for a cold
server:
```bash
curl -s -G --data-urlencode "query=drop table if exists cpu" http://127.0.0.1:9000/exec
```

## Step 6: Generate query files

Query generation is protocol-independent: always use `--format=questdb`.
Generate 1000 queries for all 16 cpu-only query types. All query files are uncompressed:
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

## Step 7: Run query benchmarks

IMPORTANT: Use 1 worker for queries. QuestDB parallelizes queries internally
(multi-threaded execution), so multiple client workers would over-subscribe CPU
and produce misleading results.

Queries can go over three transports, chosen with `--query-protocol`:

| Value | Transport | Port |
|---|---|---|
| `pg` | PostgreSQL wire (default) | 8812 |
| `http` | REST `/exec`, JSON results | 9000 |
| `qwp` | QuestDB Wire Protocol, columnar batches | 9000 |

All three run the same SQL with the same bind parameters, and the query files
are protocol-independent, so the same files feed every transport and the run can
be repeated per transport to compare them. Ask the operator which to use, or
default to `pg`, which is what QuestDB's published TSBS numbers use. Set it once:
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
The data on disk is identical whichever ingestion protocol wrote it, so the
query suite needs running only once per query transport being compared, not once
per ingestion protocol.

## Cleanup

When done benchmarking:
```bash
sudo docker rm -f questdb
rm -f /tmp/questdb-data.txt /tmp/questdb-data.qwp /tmp/questdb-queries-*.txt
```

## Reporting results

State which protocol produced each ingestion number. Never present an ILP
figure as a QWP figure or the reverse. For a `both` run, report the committed
rows/s for each and note that the ILP loader's own summary overstates its rate,
because it counts bytes handed to a socket rather than rows the server
confirmed.

## Notes

- The `cpu-only` use case with scale=4000 and 1-day window is a solid general benchmark
- For `--use-case` you can also try `devops` or `iot` (each has different query types)
- Run `tsbs_generate_queries --help` to see the full matrix of use-case + query-type combos
- Port 9000 = Web Console, QWP ingestion and QWP/HTTP queries; 9009 = ILP ingestion; 8812 = PostgreSQL wire queries
- Ingestion uses 9000 (QWP) or 9009 (ILP); queries use 8812 (`pg`) or 9000 (`http`, `qwp`)
- Ingestion protocol and query protocol are independent: data written over ILP can be queried over QWP and the reverse
- QWP keeps scaling as workers are added, where ILP flattens out once the
  server's text parsing saturates, so do not tune the worker count on an ILP
  run and reuse it for QWP
- Useful extra loader flags: `--qwp-await-ack` waits for the server to
  acknowledge every batch before counting it, `--qwp-addr` points at a
  different host or a comma-separated list for failover, and `--qwp-sf-dir`
  turns on durable store-and-forward, which trades throughput for durability
  and should be reported as a separate mode
