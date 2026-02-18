# TSBS Benchmark Agent Skill

AI coding agent skill that runs end-to-end [TSBS](https://github.com/questdb/tsbs) (Time Series Benchmark Suite) benchmarks against [QuestDB](https://questdb.io/) in Docker. Works with both [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [OpenAI Codex](https://openai.com/index/introducing-codex/).

Instead of manually running a dozen commands, just tell your agent to benchmark QuestDB and it handles everything - installing prerequisites, spinning up Docker, building TSBS, generating data, loading it, and running all 16 query types.

## What the skill does

The skill walks the agent through the full TSBS pipeline:

1. **Prerequisites** - Checks for and installs Docker, Go 1.22.5, and build tools
2. **Start QuestDB** - Pulls and runs `questdb/questdb:latest` in Docker
3. **Build TSBS** - Clones [questdb/tsbs](https://github.com/questdb/tsbs) and compiles the four required binaries
4. **Generate data** - Produces ~12 GB of uncompressed `cpu-only` data (34.5M rows, 345.6M metrics)
5. **Load data** - Ingests via ILP with auto-scaled worker count (up to 32)
6. **Generate queries** - Creates 1,000 queries for each of the 16 `cpu-only` query types
7. **Run benchmarks** - Executes all query types with a single worker (QuestDB parallelizes internally)
8. **Cleanup** - Removes the Docker container and temporary files

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
| Data size | ~12 GB uncompressed |
| Rows | 34.5M |
| Metrics | 345.6M |
| Query types | 16 |
| Queries per type | 1,000 |

**Query types benchmarked:** `cpu-max-all-1`, `cpu-max-all-8`, `cpu-max-all-32-24`, `single-groupby-1-1-1`, `single-groupby-1-1-12`, `single-groupby-1-8-1`, `single-groupby-5-1-1`, `single-groupby-5-1-12`, `single-groupby-5-8-1`, `double-groupby-1`, `double-groupby-5`, `double-groupby-all`, `high-cpu-1`, `high-cpu-all`, `lastpoint`, `groupby-orderby-limit`

## Key design decisions

- **No gzip** - Data files are kept uncompressed. Compression/decompression adds CPU overhead that skews results, especially on smaller machines.
- **Single query worker** - QuestDB parallelizes queries internally with multi-threaded execution. Multiple client workers would over-subscribe CPU and produce misleading results.
- **Auto-scaled load workers** - Data loading uses one worker per CPU core (capped at 32) since ILP ingestion benefits from client-side parallelism.

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 9000 | HTTP | Web Console |
| 9009 | TCP | ILP (line protocol) - used for data loading |
| 8812 | TCP | PostgreSQL wire - used for queries |
| 9003 | HTTP | Health/metrics |

## License

Apache 2.0
