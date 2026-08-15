---
title: "Want custom text-to-SQL?"
description: "Does a pre-generated DB profile improve text-to-SQL accuracy? No — but it 6x'd the tokens."
pubDate: "2026-08-15"
---

**TL;DR:** Just give pi/codex/claude a login and password!

I needed a small analytics page. I looked at text-to-SQL benchmarks and implemented what [BIRD](https://bird-bench.github.io/) winners [recommend](https://arxiv.org/abs/2505.19988).

Okay, but how do you test whether that actually helps? Try the newest, hottest [BEAVER benchmark](https://huggingface.co/datasets/BeaverBench/beaver). But wait: BEAVER uses (1) hand-coded Python agents, (2) reasoning=off, and (3) pre-retrieved/reranked tables (Qwen embedding top-50 / reranker top-15). This is not just firing off one prompt in chat.

So naturally, I used BEAVER's data to build my own small test.

#### Results

- **SQL execution accuracy** against the `gold` queries

| Database | pi + DB login & pass | pi + BD login & pass + Profile|
|---|---|---|
| neutron | 13% | 13% |
| nova | 9% | 8% |
| dw | 5% | 4% |

- **Input tokens grew ~6x** (pooled across runs) — the full profile is supplied again in every context, and on the biggest database it alone costs ~9x more per run than the raw arm.
- **Turns dropped from ~4.6 to ~2.2, database queries from 7.8 to 1.3.** The agent read the profile, nodded, and stopped asking questions. It was just wrong sooner.

### How I tested

[GitHub repo](https://github.com/renatyv/text2sql)

- Three MySQL database dumps from [BEAVER benchmark](https://huggingface.co/datasets/BeaverBench/beaver): neutron, nova, dw
- 100 questions per database per arm (seed-fixed sample). Real execution accuracy, scored against gold results.
- pi coding agent, GPT-5.6 Luna PRO via OpenRouter, effort=medium
- Fresh Docker container per question, a SELECT-only database account, access to the DB and the OpenRouter endpoint only
- Maximum 15 turns per query

### The third arm: metadata instead of the full profile

| Database | agent | agent + Profile | agent + Metadata |
|---|---|---|---|
| neutron | 13% | 13% | 13% |
| nova | 9% | 8% | 11% |
| dw | 5% | 4% | 3% |

Okay, maybe the full profile is too much. Let's make a summary of the profile + schema linking.
The same coding agent generated the metadata in the same sandbox from two artifacts:

1. **The [db-snooper](https://github.com/renatyv/db-snooper) profile** — against MySQL produces per-table schema (columns, types, indexes, FKs), inferred relationships, per-column value distributions, and sample rows.
2. **[Schema links](https://github.com/renatyv/schema-linker)** — deterministic linking hints: declared foreign keys from `information_schema` plus same-name column candidates (excluding noise like `id`, `created_at`), grouped into "declared" vs "inferred" join paths.

A one-off pi agent run (same Docker isolation, SELECT-only account, no access to benchmark questions or answers) read both files and wrote a short `<db>.md` with two sections: **Clarified Semantics** and **Potential Join Strategies** (join predicates with cardinality caveats). That 26–43-line file is what the `metadata` arm got. A fourth arm with both profile and metadata exists in the harness, but the headline run used three.

## Pitfalls and bad ideas encountered

- pi silently defaults to a 128k context window for unknown models
- Telling the agent to LIMIT 5 exploratory queries is BAD. The model appended LIMIT 5 to its final SQL, tanking execution accuracy.
- Restricting queries to 100 rows is BAD. Some of BEAVER's gold queries return 1000s of rows. Ouch!
- Agents never returned SQL. Fix: the prompt reserves the final turn for the answer and steers on the next to the last turn.
