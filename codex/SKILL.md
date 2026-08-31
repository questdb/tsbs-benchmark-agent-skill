---
name: questdb-tsbs-benchmark
description: Use when asked to run, compare, or troubleshoot QuestDB TSBS ingestion or query-latency benchmarks in Docker.
---

# QuestDB TSBS benchmark

Run the TSBS `cpu-only` workload against QuestDB in Docker. Keep workload inputs and benchmark policy explicit, collect each measured sample separately, and summarize the samples only after the run.

Assemble the snippets below into one `benchmark.sh` and execute them in one Bash process; later sections use variables and functions from earlier ones. Start the script with strict failure handling so a failed TSBS command cannot be hidden by `tee`:

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
```

## Benchmark choices

Use these practical defaults unless the operator requests different values:

```bash
QUESTDB_IMAGE="${QUESTDB_IMAGE:-questdb/questdb:latest}"
CONTAINER_NAME="${CONTAINER_NAME:-questdb-tsbs-benchmark}"
INGEST_PROTOCOL="${INGEST_PROTOCOL:-ilp}"       # ilp | ilp-http | qwip
QUERY_PROTOCOL="${QUERY_PROTOCOL:-pgwire}"     # pgwire | http | qwep
QUERY_CACHE="${QUERY_CACHE:-warm}"             # warm | cold
RUNS="${RUNS:-3}"
WARMUP_RUNS="${WARMUP_RUNS:-3}"
HOSTS="${HOSTS:-4000}"
QUERIES_PER_TYPE="${QUERIES_PER_TYPE:-1000}"
LOAD_WORKERS="${LOAD_WORKERS:-$(nproc)}"
METRICS_PER_ROW="${METRICS_PER_ROW:-10}"
PAGE_CACHE_RESET_COMMAND="${PAGE_CACHE_RESET_COMMAND:-}"
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

Require Linux, Docker, Git, Make, a C compiler, `curl`, `jq`, GNU core utilities (`date`, `nproc`, and `seq`), and a Go version compatible with the current `questdb/tsbs` `go.mod`.

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
RUN_ID="${RUN_ID:-$(date -u +%Y%m%dT%H%M%SZ)-$INGEST_PROTOCOL-$QUERY_PROTOCOL-$QUERY_CACHE}"
RESULTS_DIR="$WORKDIR/results/$RUN_ID"
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
load_help=$("$TSBS_BIN/tsbs_load_questdb" --help 2>&1 || true)
query_help=$("$TSBS_BIN/tsbs_run_queries_questdb" --help 2>&1 || true)
grep -Fq "$INGEST_PROTOCOL" <<<"$load_help" || {
  printf 'TSBS loader does not list protocol %s\n' "$INGEST_PROTOCOL" >&2
  exit 1
}
grep -Fq "$QUERY_PROTOCOL" <<<"$query_help" || {
  printf 'TSBS query runner does not list protocol %s\n' "$QUERY_PROTOCOL" >&2
  exit 1
}
```

## 3. Start a clean QuestDB

```bash
wait_for_questdb() {
  for _ in $(seq 1 60); do
    if curl -fsS -o /dev/null http://127.0.0.1:9000/ping; then
      return 0
    fi
    sleep 1
  done
  docker logs "$CONTAINER_NAME" >&2 || true
  return 1
}

remove_benchmark_container() {
  if ! docker container inspect "$CONTAINER_NAME" >/dev/null 2>&1; then
    return 0
  fi
  owner=$(docker inspect --format '{{ index .Config.Labels "com.questdb.tsbs-benchmark.run" }}' "$CONTAINER_NAME")
  if [ "$owner" != "$RUN_ID" ]; then
    printf 'refusing to remove unowned container %s\n' "$CONTAINER_NAME" >&2
    return 1
  fi
  docker rm -f "$CONTAINER_NAME" >/dev/null
}

start_clean_questdb() {
  remove_benchmark_container
  docker run -d --name "$CONTAINER_NAME" \
    --label "com.questdb.tsbs-benchmark.run=$RUN_ID" \
    -p 127.0.0.1:9000:9000 \
    -p 127.0.0.1:9009:9009 \
    -p 127.0.0.1:8812:8812 \
    "$QUESTDB_IMAGE" >/dev/null
  wait_for_questdb
}

docker pull "$QUESTDB_IMAGE"
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

Keep generated benchmark inputs uncompressed so compression work is outside the timed commands. The default two-day, 10-second workload produces `HOSTS * 17280` rows; set `EXPECTED_ROWS` explicitly if changing the time window or interval.

```bash
EXPECTED_ROWS="${EXPECTED_ROWS:-$((HOSTS * 17280))}"
```

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

Each ingestion sample starts with an empty database and loads the same generated file. Capture every sample separately. A sample completes only when QuestDB exposes the expected row count.

```bash
questdb_row_count() {
  curl -fsS -G --data-urlencode 'query=select count() from cpu' \
    http://127.0.0.1:9000/exec | jq -er '.dataset[0][0]'
}

wait_for_rows() {
  expected=$1
  deadline=$((SECONDS + 600))
  count=0
  while [ "$SECONDS" -lt "$deadline" ]; do
    count=$(questdb_row_count 2>/dev/null || printf '0')
    if [ "$count" -eq "$expected" ]; then
      return 0
    fi
    sleep 1
  done
  printf 'row-count timeout: expected=%s observed=%s\n' "$expected" "$count" >&2
  return 1
}

for run in $(seq 1 "$RUNS"); do
  log="$RESULTS_DIR/ingestion-run-$run.log"
  start_clean_questdb
  started_ns=$(date +%s%N)
  "$TSBS_BIN/tsbs_load_questdb" \
    --file="$DATA_FILE" \
    --workers="$LOAD_WORKERS" \
    --protocol="$INGEST_PROTOCOL" \
    2>&1 | tee "$log"
  wait_for_rows "$EXPECTED_ROWS"
  duration_ns=$(($(date +%s%N) - started_ns))
  row_rate=$(awk -v rows="$EXPECTED_ROWS" -v ns="$duration_ns" \
    'BEGIN { printf "%.3f", rows / (ns / 1000000000) }')
  metric_rate=$(awk -v rate="$row_rate" -v metrics="$METRICS_PER_ROW" \
    'BEGIN { printf "%.3f", rate * metrics }')
  printf 'applied_duration_ns=%s\napplied_rows_per_second=%s\napplied_metrics_per_second=%s\n' \
    "$duration_ns" "$row_rate" "$metric_rate" | tee -a "$log"
done
```

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

```bash
wait_for_rows "$EXPECTED_ROWS"
```

### Warm cache policy

Apply one controlled QuestDB restart and Linux page-cache reset, then perform untimed passes over every selected query type in round-robin order. Set `PAGE_CACHE_RESET_COMMAND` to a reviewed command supported by the host; there is no privileged default. Fail closed rather than labelling restart-only results as `warm` or `cold`.

```bash
if [ -z "$PAGE_CACHE_RESET_COMMAND" ]; then
  printf 'set PAGE_CACHE_RESET_COMMAND before query benchmarking\n' >&2
  exit 1
fi

reset_query_cache_state() {
  docker restart "$CONTAINER_NAME" >/dev/null
  wait_for_questdb
  bash -c "$PAGE_CACHE_RESET_COMMAND"
}

if [ "$QUERY_CACHE" = "warm" ]; then
  reset_query_cache_state

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
      reset_query_cache_state
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
remove_benchmark_container
```

Keep `$RESULTS_DIR` unless the operator explicitly asks to remove it.
