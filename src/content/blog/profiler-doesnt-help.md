---
title: "Want custom text-to-SQL?"
description: "Does a pre-generated DB profile improve text-to-SQL accuracy? No — but it 6x'd the tokens."
pubDate: "2026-08-15"
---

**TL;DR:** Give pi, Codex, or Claude a database login and password. A full database profile did not improve execution accuracy (25/300 correct versus 27/300) and used about 6× more input tokens.

I needed a small analytics page with text-to-SQL, so I looked at the benchmarks and implemented what [BIRD](https://bird-bench.github.io/) winners [recommend](https://arxiv.org/abs/2505.19988): give the model a pre-generated database profile.

But does that profile actually help? I tested it on the new [BEAVER benchmark](https://huggingface.co/datasets/BeaverBench/beaver). There is a catch: BEAVER uses hand-coded Python agents, turns reasoning off, and pre-retrieves and reranks tables (Qwen embedding top 50, reranker top 15).

That raised a simpler question: what happens when an agent can query the database directly? So I tested that too.

**Experiment:** 300 seed-fixed [BEAVER](https://huggingface.co/datasets/BeaverBench/beaver) questions per arm: 100 each from neutron, nova, and dw. The primary metric is **execution accuracy**: the share of generated SQL queries whose result matches the gold query. Higher is better.

## Results at a glance

| Database | Raw DB access | Full profile | Change | What it means |
|---|---:|---:|---:|---|
| neutron | 13/100 (13%) | 13/100 (13%) | 0 pp | No change |
| nova | 9/100 (9%) | 8/100 (8%) | −1 pp | No improvement |
| dw | 5/100 (5%) | 4/100 (4%) | −1 pp | No improvement |
| **Overall** | **27/300 (9.0%)** | **25/300 (8.3%)** | **−0.7 pp** | **No aggregate gain** |

The profile changed cost and behavior, not correctness:

| Metric | Raw DB access | Full profile | Change | What it means |
|---|---:|---:|---:|---|
| Execution accuracy ↑ | 27/300 (9.0%) | 25/300 (8.3%) | −0.7 pp | No improvement |
| Input tokens/question ↓ | 1× baseline | about 6× | about +500% | Much larger prompts |
| Turns/question ↓ | 4.6 | 2.2 | −52% | The agent stopped sooner |
| DB queries/question ↓ | 7.8 | 1.3 | −83% | The agent explored less |

<p class="result-note"><strong>Bottom line:</strong> the agent read the profile, asked fewer questions, and reached the wrong answer faster.</p>

The full profile was included in every context. On the largest database, the profile text alone added about nine times the direct-access arm's input-token volume per run.

### How I tested

[GitHub repo](https://github.com/renatyv/text2sql)

- Three MySQL database dumps from [BEAVER benchmark](https://huggingface.co/datasets/BeaverBench/beaver): neutron, nova, dw
- 100 questions per database per arm, using a seed-fixed sample. I measured execution accuracy against the gold results.
- pi coding agent with GPT-5.6 Luna PRO via OpenRouter at medium effort
- A fresh Docker container per question, with a SELECT-only database account and access only to the database and OpenRouter endpoint
- At most 15 turns per query

### The third arm: metadata instead of the full profile

| Database | Raw DB access | Full profile | Compact metadata | Metadata change vs raw |
|---|---:|---:|---:|---:|
| neutron | 13/100 (13%) | 13/100 (13%) | 13/100 (13%) | 0 pp |
| nova | 9/100 (9%) | 8/100 (8%) | 11/100 (11%) | +2 pp |
| dw | 5/100 (5%) | 4/100 (4%) | 3/100 (3%) | −2 pp |
| **Overall** | **27/300 (9.0%)** | **25/300 (8.3%)** | **27/300 (9.0%)** | **0 pp** |

Maybe the full profile is simply too much context. For a third arm, I gave the agent a compact summary built from the profile and schema links. It matched raw access overall, but the per-database movements were only a few questions and are descriptive, not evidence of an improvement.

The same coding agent generated that metadata in the same sandbox from two artifacts:

1. **The [db-snooper](https://github.com/renatyv/db-snooper) profile** — against MySQL produces per-table schema (columns, types, indexes, FKs), inferred relationships, per-column value distributions, and sample rows.
2. **[Schema links](https://github.com/renatyv/schema-linker)** — deterministic linking hints: declared foreign keys from `information_schema` plus same-name column candidates (excluding noise like `id`, `created_at`), grouped into "declared" vs "inferred" join paths.

A one-off pi agent run—using the same Docker isolation and SELECT-only account, with no access to benchmark questions or answers—read both files and wrote a short `<db>.md`. It contained **Clarified Semantics** and **Potential Join Strategies**, including join predicates and cardinality caveats. That 26–43-line file was the input for the `metadata` arm. The harness also has a fourth arm with both the profile and metadata, but the headline run used three.

## Pitfalls and bad ideas encountered

- pi silently defaults to a 128k context window for unknown models.
- Do not tell the agent to add `LIMIT 5` to exploratory queries. It may append that limit to its final SQL, which tanks execution accuracy.
- Do not cap queries at 100 rows. Some BEAVER gold queries return thousands of rows.
- The agents sometimes never returned SQL. Reserving the final turn for the answer, with a reminder on the penultimate turn, fixed it.
