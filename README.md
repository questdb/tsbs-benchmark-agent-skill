# TSBS Benchmark Agent Skill

AI coding agent skill that runs end-to-end [TSBS](https://github.com/questdb/tsbs) (Time Series Benchmark Suite) benchmarks against [QuestDB](https://questdb.io/) in Docker. Works with both [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [OpenAI Codex](https://openai.com/index/introducing-codex/).

Instead of manually running a dozen commands, just tell your agent to benchmark QuestDB and it handles everything - installing prerequisites, spinning up Docker, building TSBS, generating data, loading it, and running all 16 query types.

Ingestion can be benchmarked over either of QuestDB's two protocols: **QWP**, the binary columnar QuestDB Wire Protocol carried on a WebSocket, or **ILP**, InfluxDB line protocol text over TCP.

## What the skill does

The skill walks the agent through the full TSBS pipeline:

1. **Choose the protocol** - Asks whether to benchmark `qwp`, `ilp-http`, `ilp`, or `all`, defaulting to `qwp` when unattended
2. **Prerequisites** - Checks for and installs Docker, Go 1.23+ (current stable, right architecture for x86 or Graviton), and build tools
3. **Start QuestDB** - Pulls and runs `questdb/questdb:nightly` in Docker
4. **Build TSBS** - Clones [questdb/tsbs](https://github.com/questdb/tsbs) and compiles the four required binaries
5. **Confirm QWP** - Loads ten rows over QWP first, so a server that does not speak it is caught in seconds rather than after gigabytes of data
6. **Generate data** - Produces uncompressed `cpu-only` data (34.5M rows, 345.6M metrics): ~3.6 GB binary for QWP, ~12 GB text for ILP
7. **Load data** - Ingests over the chosen protocol with auto-scaled worker count (up to 32), then confirms the committed row count from the server
8. **Generate queries** - Creates 1,000 queries for each of the 16 `cpu-only` query types
9. **Run benchmarks** - Executes all query types with a single worker (QuestDB parallelizes internally) over the chosen query transport: `pg`, `http` or `qwp`. TSBS measures reads as well as writes; skip this phase if only ingestion throughput is wanted
10. **Cleanup** - Removes the Docker container and temporary files

## Ingestion protocols

| | QWP | ILP over HTTP | ILP over TCP |
|---|---|---|---|
| Shape | binary, columnar | line protocol text | line protocol text |
| Transport | WebSocket, port 9000 | HTTP, port 9000 | TCP, port 9009 |
| Generator format | `questdb-qwp` | `questdb` | `questdb` |
| Loader flag | `--protocol=qwp` (default) | `--protocol=ilp-http` | `--protocol=ilp` |
| Delivery | acknowledged by the server | server processed the batch | fire and forget |
| Data size at scale 4000 | ~3.6 GB | ~12 GB | ~12 GB |

QWP is the faster path and the one to use unless a line protocol baseline is wanted. For that baseline use **`ilp-http`**, the transport QuestDB recommends for line protocol today.

**An `ilp` (TCP) number depends on the server's thread pools.** On a 32 vCPU r8a.8xlarge with 69.1M rows and 32 workers, ILP/TCP sent 9.2M rows/s on the 9.4.3 release with defaults, 1.7M on the 9.4.4-SNAPSHOT nightly with defaults, and 8.0M on that same nightly with `QDB_LINE_TCP_IO_WORKER_COUNT=16`. The nightly gives the ILP/TCP pools 2 threads where the shared pools each get 31. The skill checks the pool sizes and records them with the result; `ilp-http` rides the shared pools and needed no tuning on either build.

Reporting honestly matters too: a QWP run's row count is what the server confirmed, whereas the `ilp` loader summary counts bytes written to a socket. The skill polls the server for the committed row count after every load, which is the only figure comparable across transports. That gap is real: QWP sends at 11.5M rows/s without per-batch acks and 14.3M with them, yet commits 6.7M and 6.3M respectively - the send rate moves 25% depending purely on when the client waits.

The QWP data format matters as much as the protocol. `questdb-qwp` writes the same points in a binary schema-and-dictionary encoding, so the loader sends them without parsing text. Loading text over QWP works but measures the client's parser as much as the database.

## Query transports

Reads have their own transport, independent of how the data was written, selected with `--query-protocol`:

| Value | Transport | Port |
|---|---|---|
| `pg` | PostgreSQL wire (pgx v5), the default | 8812 |
| `http` | REST `/exec`, JSON results | 9000 |
| `qwp` | QuestDB Wire Protocol, columnar result batches | 9000 |

All three send identical SQL with identical bind parameters, and the query files are protocol-independent, so the suite can be re-run per transport to compare them directly. QWP streams results as columnar batches instead of JSON, which shows up most on queries returning many rows or many groups.

## Repository structure

```
claude/
  SKILL.md          # Skill definition for Claude Code
codex/
  SKILL.md          # Skill definition for OpenAI Codex
  agents/
    openai.yaml     # Codex agent configuration
```

## Usage

### Claude Code

Copy the skill into your Claude Code skills directory:

```bash
cp -r claude/SKILL.md ~/.claude/skills/tsbs-benchmark/SKILL.md
```

Then in Claude Code, ask it to run the TSBS benchmark against QuestDB.

### OpenAI Codex

Use the Codex agent definition and skill together. The `codex/agents/openai.yaml` provides the agent interface, and `codex/SKILL.md` provides the benchmark instructions.

## Benchmark details

| Parameter | Value |
|---|---|
| Use case | `cpu-only` |
| Scale | 4,000 hosts |
| Time window | 1 day (2016-01-01) |
| Log interval | 10s |
| Data size | ~3.6 GB (QWP) / ~12 GB (ILP), uncompressed |
| Rows | 34.5M |
| Metrics | 345.6M |
| Query types | 16 |
| Queries per type | 1,000 |

**Query types benchmarked:** `cpu-max-all-1`, `cpu-max-all-8`, `cpu-max-all-32-24`, `single-groupby-1-1-1`, `single-groupby-1-1-12`, `single-groupby-1-8-1`, `single-groupby-5-1-1`, `single-groupby-5-1-12`, `single-groupby-5-8-1`, `double-groupby-1`, `double-groupby-5`, `double-groupby-all`, `high-cpu-1`, `high-cpu-all`, `lastpoint`, `groupby-orderby-limit`

## Key design decisions

- **No gzip** - Data files are kept uncompressed. Compression/decompression adds CPU overhead that skews results, especially on smaller machines.
- **Single query worker** - QuestDB parallelizes queries internally with multi-threaded execution. Multiple client workers would over-subscribe CPU and produce misleading results.
- **Auto-scaled load workers** - Data loading uses one worker per CPU core (capped at 32) since both protocols benefit from client-side parallelism. QWP keeps scaling further than ILP, which flattens out once the server's text parsing saturates, so a worker count tuned on an ILP run should not be reused for QWP.
- **Native format per protocol** - A QWP run loads the binary `questdb-qwp` format rather than text, so the benchmark measures ingestion instead of the client's line parser.
- **Committed rows, not bytes sent** - After loading, the skill queries the server for the row count, because the ILP path returns as soon as bytes reach the socket and the server may still be applying rows.
- **Nightly image** - QWP support ships in `questdb/questdb:nightly`, and a smoke load verifies it before any large data set is generated.

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 9000 | HTTP / WebSocket | Web Console, QWP ingestion, and `http` / `qwp` queries |
| 9009 | TCP | ILP (line protocol) - used for data loading |
| 8812 | TCP | PostgreSQL wire - used for `pg` queries (the default) |
| 9003 | HTTP | Health/metrics |

## License

Apache 2.0
