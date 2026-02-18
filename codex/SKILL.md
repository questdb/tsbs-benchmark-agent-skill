---
name: questdb-tsbs-benchmark
description: Run full TSBS (Time Series Benchmark Suite) benchmarks against QuestDB in Docker, including prerequisite checks, TSBS build, data generation, loading, query generation, and benchmark execution. Use when asked to set up or run QuestDB TSBS performance tests end-to-end.
---

# TSBS Benchmark for QuestDB

Run the full TSBS (Time Series Benchmark Suite) against QuestDB running in Docker.
Handle all prerequisites, data generation, loading, and query benchmarking.

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
Check if Go is installed:
```bash
go version
```
If Go is not installed, install Go 1.22.5:
```bash
curl -fsSL https://go.dev/dl/go1.22.5.linux-amd64.tar.gz -o /tmp/go.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf /tmp/go.tar.gz && rm /tmp/go.tar.gz
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
```

### 3. Build tools
```bash
sudo apt-get install -y -qq make gcc gzip
```

## Step 1: Start QuestDB in Docker

Stop and remove any existing QuestDB container first to start clean:
```bash
sudo docker rm -f questdb 2>/dev/null || true
```
Pull latest image and start:
```bash
sudo docker pull questdb/questdb:latest
sudo docker run -d --name questdb \
  -p 9000:9000 -p 9009:9009 -p 8812:8812 -p 9003:9003 \
  questdb/questdb:latest
```
Verify it is running:
```bash
sudo docker ps --filter name=questdb --format '{{.Status}}'
```

## Step 2: Clone And Build TSBS

```bash
cd /home/ubuntu
git clone https://github.com/questdb/tsbs.git
cd /home/ubuntu/tsbs
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
make tsbs_generate_data tsbs_generate_queries tsbs_load_questdb tsbs_run_queries_questdb
```
Build only the four binaries needed for QuestDB benchmarking into `bin/`.

## Step 3: Generate Benchmark Data

IMPORTANT: Do not compress (`gzip`) the data file. Compression/decompression adds CPU overhead that skews benchmark results, especially on smaller machines.

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
This generates approximately 12GB of uncompressed data (34.5M rows, 345.6M metrics).

## Step 4: Load Data Into QuestDB

Use as many workers as CPU cores available (up to 32). Capture the load summary and report ingestion throughput in rows/s (and metrics/s):
```bash
WORKERS=$(nproc)
if [ "$WORKERS" -gt 32 ]; then WORKERS=32; fi

/home/ubuntu/tsbs/bin/tsbs_load_questdb \
  --file=/tmp/questdb-data.txt \
  --workers=$WORKERS \
  2>&1 | tee /tmp/questdb-load.log

# Report ingestion throughput from TSBS summary
# Example output line includes: "overall row/s"
grep -E "loaded .*metrics|loaded .*rows" /tmp/questdb-load.log
```
Expected summary lines include the mean rates, e.g. `overall row/s` for rows/second and `overall metric/s` for metrics/second.

## Step 5: Generate Query Files

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

## Step 6: Run Query Benchmarks

IMPORTANT: Use one worker for queries. QuestDB parallelizes queries internally (multi-threaded execution), so multiple client workers over-subscribe CPU and produce misleading results.

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
    --print-interval=0
  echo ""
done
```

## Cleanup

When done benchmarking:
```bash
sudo docker rm -f questdb
rm -f /tmp/questdb-data.txt /tmp/questdb-queries-*.txt
```

## Notes

- `cpu-only` with `scale=4000` and a 1-day window is a solid general benchmark.
- For `--use-case`, also try `devops` or `iot` (each has different query types).
- Run `tsbs_generate_queries --help` to see the full matrix of use-case + query-type combinations.
- Port `9000` is Web Console (HTTP), `9009` is ILP (line protocol), and `8812` is PostgreSQL wire.
- The data loading step uses ILP (`9009`); queries use PostgreSQL wire (`8812`).
