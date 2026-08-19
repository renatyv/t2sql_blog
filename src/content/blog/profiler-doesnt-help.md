---
title: "Want custom text-to-SQL?"
description: "Does a pre-generated DB profile improve text-to-SQL accuracy? No — but it 6x'd the tokens."
pubDate: "2026-08-15"
---

**TL;DR:** Give pi, Codex, or Claude a database login and password.

I needed a small analytics page with text-to-SQL, so I looked at the benchmarks and implemented what [BIRD](https://bird-bench.github.io/) winners [recommend](https://arxiv.org/abs/2505.19988): give the model a pre-generated database profile.

But does that profile actually help? I tested it on the new [BEAVER benchmark](https://huggingface.co/datasets/BeaverBench/beaver). There is a catch: BEAVER uses hand-coded Python agents, turns reasoning off, and pre-retrieves and reranks tables (Qwen embedding top 50, reranker top 15).

That raised a simpler question: what happens when agent can query the database directly? So I tested that too.

#### Results

- **SQL accuracy have not improved** against "gold" queries.

| Database | pi + DB login & password | pi + DB login & password + profile |
|---|---|---|
| neutron | 13% | 13% |
| nova | 9% | 8% |
| dw | 5% | 4% |

- **Input tokens grew by about 6×** across runs. The full profile is included in every context; on the largest database, it alone cost about 9× more per run than the direct-access arm.
- **Turns fell from about 4.6 to 2.2, and database queries from 7.8 to 1.3.** The agent read the profile, stopped asking questions, and reached the wrong answer faster.

### How I tested

[GitHub repo](https://github.com/renatyv/text2sql)

- Three MySQL database dumps from [BEAVER benchmark](https://huggingface.co/datasets/BeaverBench/beaver): neutron, nova, dw
- 100 questions per database per arm, using a seed-fixed sample. I measured execution accuracy against the gold results.
- pi coding agent with GPT-5.6 Luna PRO via OpenRouter at medium effort
- A fresh Docker container per question, with a SELECT-only database account and access only to the database and OpenRouter endpoint
- At most 15 turns per query

### The third arm: metadata instead of the full profile

| Database | agent | agent + Profile | agent + Metadata |
|---|---|---|---|
| neutron | 13% | 13% | 13% |
| nova | 9% | 8% | 11% |
| dw | 5% | 4% | 3% |

Maybe the full profile is simply too much context. For a third arm, I gave the agent a compact summary built from the profile and schema links.

The same coding agent generated that metadata in the same sandbox from two artifacts:

1. **The [db-snooper](https://github.com/renatyv/db-snooper) profile** — against MySQL produces per-table schema (columns, types, indexes, FKs), inferred relationships, per-column value distributions, and sample rows.
2. **[Schema links](https://github.com/renatyv/schema-linker)** — deterministic linking hints: declared foreign keys from `information_schema` plus same-name column candidates (excluding noise like `id`, `created_at`), grouped into "declared" vs "inferred" join paths.

A one-off pi agent run—using the same Docker isolation and SELECT-only account, with no access to benchmark questions or answers—read both files and wrote a short `<db>.md`. It contained **Clarified Semantics** and **Potential Join Strategies**, including join predicates and cardinality caveats. That 26–43-line file was the input for the `metadata` arm. The harness also has a fourth arm with both the profile and metadata, but the headline run used three.

## Pitfalls and bad ideas encountered

- pi silently defaults to a 128k context window for unknown models.
- Do not tell the agent to add `LIMIT 5` to exploratory queries. It may append that limit to its final SQL, which tanks execution accuracy.
- Do not cap queries at 100 rows. Some BEAVER gold queries return thousands of rows.
- The agents sometimes never returned SQL. Reserving the final turn for the answer, with a reminder on the penultimate turn, fixed it.
