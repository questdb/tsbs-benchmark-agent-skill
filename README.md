# TSBS Benchmark Agent Skill

AI coding agent skill for running end-to-end [TSBS](https://github.com/questdb/tsbs) ingestion and query-latency benchmarks against [QuestDB](https://questdb.io/) in Docker. It works with both [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [OpenAI Codex](https://openai.com/index/introducing-codex/).

Tell the agent to benchmark QuestDB and it handles the workflow: prerequisites, Docker setup, TSBS build, workload generation, ingestion, query preparation, repeated measurements, result summaries, and cleanup.

## Methodology

The skill follows the same broad TSBS workflow used for QuestDB benchmark runs:

1. Choose ingestion and query protocols.
2. Generate the `cpu-only` dataset and query streams once from explicit inputs.
3. Measure ingestion repeatedly from an empty database.
4. Prepare query latency on a fresh database and load the dataset once outside the timed query runs.
5. Apply one cache policy for the full query run:
   - `warm`: restart and reset the Linux page cache once, then run untimed round-robin warm-up passes before measurement;
   - `cold`: restart and reset the Linux page cache before each timed query job.
6. Run each query type with one query worker and collect multiple measured samples.
7. Keep raw per-run output and summarize the samples.

The Docker workflow stays intentionally approachable. Workload size, worker allocation, core pinning, protocols, cache policy, warm-up depth, and sample count can all be adjusted for the machine and benchmark goal.

## Protocols

Ingestion and query transport are selected independently:

| Phase | Options |
| --- | --- |
| Ingestion | ILP over TCP (`ilp`), ILP over HTTP (`ilp-http`), or QuestDB Wire Ingestion Protocol (`qwip`) |
| Query latency | PostgreSQL wire (`pgwire`), REST (`http`), or QuestDB Wire Execution Protocol (`qwep`) |

QWIP data uses TSBS's binary `questdb-qwp` generator format. ILP uses the text `questdb` format. Query streams use `questdb` for every query transport.

## Example defaults

| Parameter | Value |
| --- | --- |
| Use case | `cpu-only` |
| Scale | 4,000 hosts |
| Time window | 2 days |
| Log interval | 10 seconds |
| Seed | 123 |
| Query types | 16 |
| Queries per type | 1,000 |
| Measured samples | 3 |
| Query workers | 1 |
| Query cache | warm, with 3 untimed passes by default |

The defaults are a starting point, not a required benchmark profile.

## Repository structure

```text
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
mkdir -p ~/.claude/skills/tsbs-benchmark
cp claude/SKILL.md ~/.claude/skills/tsbs-benchmark/SKILL.md
```

Then ask Claude Code to run a QuestDB TSBS benchmark. Mention any protocol, workload, cache, or sample-count preferences; otherwise the skill uses its practical defaults.

### OpenAI Codex

Copy `codex/SKILL.md` into the Codex skills directory and use it with `codex/agents/openai.yaml`.

## Results

The skill retains raw logs and per-query JSON, then reports:

- workload inputs and selected benchmark policy;
- ingestion rows/s and metrics/s for each measured sample;
- per-query QPS and latency quantiles for each measured sample;
- mean, minimum, maximum, and population standard deviation across successful samples;
- failed or incomplete samples without silently excluding them.

Warm-up output is kept separate from measured results.

## Ports

| Port | Protocol | Purpose |
| --- | --- | --- |
| 9000 | HTTP / WebSocket | Web Console, ILP over HTTP, QWIP, REST queries, and QWEP |
| 9009 | TCP | ILP over TCP |
| 8812 | TCP | PostgreSQL wire queries |
| 9003 | HTTP | Health and metrics |

## License

Apache 2.0
