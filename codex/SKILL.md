---
name: questdb-tsbs-benchmark
description: Use when asked to run, compare, or troubleshoot QuestDB TSBS ingestion or query-latency benchmarks in Docker.
---

# QuestDB TSBS benchmark

Run the TSBS `cpu-only` workload against QuestDB in Docker. Keep workload inputs and benchmark policy explicit, collect each measured sample separately, and summarize the samples only after the run.

## Benchmark choices

Use these practical defaults unless the operator requests different values:

```bash
QUESTDB_IMAGE="${QUESTDB_IMAGE:-questdb/questdb:latest}"
INGEST_PROTOCOL="${INGEST_PROTOCOL:-ilp}"       # ilp | ilp-http | qwip
QUERY_PROTOCOL="${QUERY_PROTOCOL:-pgwire}"     # pgwire | http | qwep
QUERY_CACHE="${QUERY_CACHE:-warm}"             # warm | cold
RUNS="${RUNS:-3}"
WARMUP_RUNS="${WARMUP_RUNS:-3}"
HOSTS="${HOSTS:-4000}"
QUERIES_PER_TYPE="${QUERIES_PER_TYPE:-1000}"
LOAD_WORKERS="${LOAD_WORKERS:-$(nproc)}"
if [ "$LOAD_WORKERS" -gt 32 ]; then LOAD_WORKERS=32; fi
```

Ingestion and query protocols are independent. Use the protocol names exposed by the checked-out TSBS binaries:

| Phase | Choice | Transport |
| --- | --- | --- |
| Ingestion | `ilp` | line protocol over TCP, port 9009 |
| Ingestion | `ilp-http` | line protocol over HTTP, port 9000 |
| Ingestion | `qwip` | QuestDB Wire Ingestion Protocol, port 9000 |
| Query | `pgwire` | PostgreSQL wire, port 8812 |
| Query | `http` | REST, port 9000 |
| Query | `qwep` | QuestDB Wire Execution Protocol, port 9000 |

Choose protocols before generating data. `qwip` uses the `questdb-qwp` data format; the ILP transports use `questdb`. Query files always use `questdb` and can be reused across query protocols. `RUNS` is the sample count for ingestion and for each selected query type.

## 1. Check prerequisites

Require Linux, Docker, Git, Make, a C compiler, `curl`, and a Go version compatible with the current `questdb/tsbs` `go.mod`.

```bash
docker --version
docker info --format '{{.ServerVersion}}'
go version
git --version
```

Install missing tools using the host's supported package manager. Ensure there is enough free disk for generated data, query files, Docker storage, and result logs.

## 2. Build TSBS

Use a dedicated workspace and the current QuestDB TSBS fork:

```bash
WORKDIR="${WORKDIR:-$HOME/tsbs-benchmark}"
TSBS_DIR="$WORKDIR/tsbs"
RESULTS_DIR="$WORKDIR/results"
mkdir -p "$WORKDIR" "$RESULTS_DIR"

if [ ! -d "$TSBS_DIR/.git" ]; then
  git clone https://github.com/questdb/tsbs.git "$TSBS_DIR"
else
  git -C "$TSBS_DIR" pull --ff-only
fi

make -C "$TSBS_DIR" \
  tsbs_generate_data \
  tsbs_generate_queries \
  tsbs_load_questdb \
  tsbs_run_queries_questdb

TSBS_BIN="$TSBS_DIR/bin"
```

Before a large run, check that the selected protocols are supported:

```bash
"$TSBS_BIN/tsbs_load_questdb" --help | grep -E 'protocol|qwip|ilp-http'
"$TSBS_BIN/tsbs_run_queries_questdb" --help | grep -E 'query-protocol|qwep|pgwire'
```

## 3. Start a clean QuestDB

```bash
wait_for_questdb() {
  until curl -fsS -o /dev/null http://127.0.0.1:9000/ping; do
    sleep 1
  done
}

start_clean_questdb() {
  docker rm -f questdb >/dev/null 2>&1 || true
  docker run -d --name questdb \
    -p 9000:9000 -p 9009:9009 -p 8812:8812 -p 9003:9003 \
    "$QUESTDB_IMAGE" >/dev/null
  wait_for_questdb
}

docker pull "$QUESTDB_IMAGE"
start_clean_questdb
```

Optional CPU pinning is fine for a colocated benchmark. If used, apply the same client/server allocation to every measured sample.

## 4. Generate workload inputs once

Use explicit inputs for both ingestion and query generation. The example uses a two-day half-open window, a 10-second interval, seed 123, and all 16 `cpu-only` query types.

```bash
START="2016-01-01T00:00:00Z"
END="2016-01-03T00:00:00Z"
INTERVAL="10s"
SEED="123"

if [ "$INGEST_PROTOCOL" = "qwip" ]; then
  DATA_FORMAT="questdb-qwp"
  DATA_FILE="$WORKDIR/questdb-data.qwp"
else
  DATA_FORMAT="questdb"
  DATA_FILE="$WORKDIR/questdb-data.txt"
fi

"$TSBS_BIN/tsbs_generate_data" \
  --use-case=cpu-only \
  --seed="$SEED" \
  --scale="$HOSTS" \
  --timestamp-start="$START" \
  --timestamp-end="$END" \
  --log-interval="$INTERVAL" \
  --format="$DATA_FORMAT" \
  > "$DATA_FILE"
```

Keep generated benchmark inputs uncompressed so compression work is outside the timed commands.

Generate query streams once:

```bash
QUERY_TYPES=(
  cpu-max-all-1 cpu-max-all-8 cpu-max-all-32-24
  single-groupby-1-1-1 single-groupby-1-1-12 single-groupby-1-8-1
  single-groupby-5-1-1 single-groupby-5-1-12 single-groupby-5-8-1
  double-groupby-1 double-groupby-5 double-groupby-all
  high-cpu-1 high-cpu-all lastpoint groupby-orderby-limit
)

for query_type in "${QUERY_TYPES[@]}"; do
  "$TSBS_BIN/tsbs_generate_queries" \
    --use-case=cpu-only \
    --seed="$SEED" \
    --scale="$HOSTS" \
    --timestamp-start="$START" \
    --timestamp-end="$END" \
    --queries="$QUERIES_PER_TYPE" \
    --query-type="$query_type" \
    --format=questdb \
    > "$WORKDIR/queries-$query_type.txt"
done
```

## 5. Measure ingestion

Each ingestion sample starts with an empty database and loads the same generated file. Capture every sample separately.

```bash
for run in $(seq 1 "$RUNS"); do
  start_clean_questdb
  "$TSBS_BIN/tsbs_load_questdb" \
    --file="$DATA_FILE" \
    --workers="$LOAD_WORKERS" \
    --protocol="$INGEST_PROTOCOL" \
    2>&1 | tee "$RESULTS_DIR/ingestion-run-$run.log"
done
```

Before accepting an ingestion sample, query QuestDB until the expected generated row count is visible. Include that wait in an end-to-end database-applied throughput measurement when comparing ingestion protocols; the loader summary alone may describe client-side completion.

## 6. Prepare query latency

Use a fresh database for query benchmarking. Load the dataset once as untimed setup, then keep it for the query samples.

```bash
start_clean_questdb
"$TSBS_BIN/tsbs_load_questdb" \
  --file="$DATA_FILE" \
  --workers="$LOAD_WORKERS" \
  --protocol="$INGEST_PROTOCOL" \
  2>&1 | tee "$RESULTS_DIR/query-setup-ingestion.log"
```

Wait for the full generated row count before running queries.

### Warm cache policy

Apply one controlled QuestDB restart and Linux page-cache reset, then perform untimed passes over every selected query type in round-robin order. Use the host's supported cache-reset mechanism. Do not mix warm-up output with measured results.

```bash
if [ "$QUERY_CACHE" = "warm" ]; then
  docker restart questdb >/dev/null
  wait_for_questdb
  # Reset the Linux page cache here using the host's supported mechanism.

  for warmup in $(seq 1 "$WARMUP_RUNS"); do
    for query_type in "${QUERY_TYPES[@]}"; do
      "$TSBS_BIN/tsbs_run_queries_questdb" \
        --file="$WORKDIR/queries-$query_type.txt" \
        --workers=1 \
        --print-interval=0 \
        --query-protocol="$QUERY_PROTOCOL" \
        > "$RESULTS_DIR/warmup-$warmup-$query_type.log" 2>&1
    done
  done
fi
```

### Cold cache policy

For `QUERY_CACHE=cold`, skip the warm-up loop and perform a controlled QuestDB restart plus Linux page-cache reset before every timed query job. Keep that policy unchanged for the entire run.

## 7. Measure query latency

Use one query worker so each TSBS process issues one query at a time. Run repetition-major: every selected query type for run 1, then every type for run 2, and so on.

```bash
for run in $(seq 1 "$RUNS"); do
  for query_type in "${QUERY_TYPES[@]}"; do
    if [ "$QUERY_CACHE" = "cold" ]; then
      docker restart questdb >/dev/null
      wait_for_questdb
      # Reset the Linux page cache here using the host's supported mechanism.
    fi

    "$TSBS_BIN/tsbs_run_queries_questdb" \
      --file="$WORKDIR/queries-$query_type.txt" \
      --workers=1 \
      --print-interval=0 \
      --query-protocol="$QUERY_PROTOCOL" \
      --results-file="$RESULTS_DIR/query-$query_type-run-$run.json" \
      2>&1 | tee "$RESULTS_DIR/query-$query_type-run-$run.log"
  done
done
```

## 8. Report results

Retain raw logs and per-query JSON. Summarize:

- workload inputs: use case, seed, hosts, time window, interval, and queries per type;
- QuestDB image, TSBS revision, protocols, worker counts, cache policy, warm-up count, and measured run count;
- each ingestion sample's database-applied rows/s and metrics/s;
- each query type's per-run QPS and latency quantiles;
- mean, minimum, maximum, and population standard deviation across successful measured samples.

Do not include warm-up output in aggregates. Call out failed or incomplete samples instead of silently dropping them.

## Cleanup

```bash
docker rm -f questdb >/dev/null 2>&1 || true
```

Keep `$RESULTS_DIR` unless the operator explicitly asks to remove it.
