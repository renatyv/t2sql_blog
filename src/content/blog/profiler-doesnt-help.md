---
title: "Your SQL Agent Doesn't Need a Database Summary"
description: "A three-arm BEAVER benchmark of raw database access, a full profile, and compact metadata."
pubDate: "2026-08-15"
---

**TL;DR:** Start a SQL agent with direct database access. In this experiment, neither a full database profile nor compact metadata improved execution accuracy. The full profile also used about 6× more input tokens and made the agent explore the database 83% less often.

I expected pre-generated metadata to save the agent from rediscovering tables, joins, data types, and values. [BIRD](https://bird-bench.github.io/) winners [recommend this approach](https://arxiv.org/abs/2505.19988), so I tested three ways of giving the agent database context:

1. **Raw database access:** the agent received no pre-generated metadata and explored the database with SQL.
2. **Full profile:** the agent received the complete db-snooper report in every context, including schemas, indexes, relationships, value distributions, and sample rows.
3. **Compact metadata:** the agent received a 26–43-line summary generated from the full profile and schema links. It kept clarified semantics and likely join strategies while omitting most profile detail.

The question was simple: **does extra database context help a coding agent produce more correct SQL?** In this setup, the answer was no. Three results support that conclusion:

1. A full profile scored 25/300, versus 27/300 with raw database access.
2. The profile made the agent cheaper in database calls but far more expensive in prompt tokens.
3. Compact metadata matched raw access at 27/300, but did not beat it.

## 1. The full profile did not improve accuracy

BEAVER is the benchmark dataset; `neutron`, `nova`, and `dw` are three MySQL databases included in it. I gave the agent 300 seed-fixed questions per arm—100 questions associated with each database. The primary metric was **execution accuracy**—whether the generated SQL returned the same result as the reference query.

| Database | Raw DB access | Full profile | Change |
|---|---:|---:|---:|
| neutron | 13/100 (13%) | 13/100 (13%) | 0 pp |
| nova | 9/100 (9%) | 8/100 (8%) | −1 pp |
| dw | 5/100 (5%) | 4/100 (4%) | −1 pp |
| **Overall** | **27/300 (9.0%)** | **25/300 (8.3%)** | **−0.7 pp** |

The profile produced no aggregate gain. BEAVER's reference setup is different: it uses hand-coded Python agents, disables reasoning, and pre-retrieves and reranks tables with Qwen embeddings (top 50) and a reranker (top 15).

## 2. The profile changed cost and behavior, not correctness

| Metric | Raw DB access | Full profile | Change |
|---|---:|---:|---:|
| Execution accuracy ↑ | 27/300 (9.0%) | 25/300 (8.3%) | −0.7 pp |
| Input tokens/question ↓ | 1× baseline | about 6× | about +500% |
| Turns/question ↓ | 4.6 | 2.2 | −52% |
| DB queries/question ↓ | 7.8 | 1.3 | −83% |

The agent read the profile, asked fewer questions, and reached the wrong answer faster. On the largest database, the profile text alone added about nine times the raw-access arm's input-token volume per run.

## 3. Compact metadata matched raw access, but did not beat it

Maybe the full profile was simply too much context. I added a third arm with a 26–43-line summary generated from two artifacts:

- A [db-snooper](https://github.com/renatyv/db-snooper) profile containing schema, indexes, foreign keys, inferred relationships, value distributions, and samples.
- [Schema links](https://github.com/renatyv/schema-linker) containing declared foreign keys and same-name join candidates, grouped into declared and inferred paths.

The same coding agent generated each summary in the same sandbox, with no access to benchmark questions or answers. The files described clarified semantics and potential join strategies, including predicates and cardinality caveats.

| Database | Raw DB access | Full profile | Compact metadata | Metadata vs raw |
|---|---:|---:|---:|---:|
| neutron | 13/100 (13%) | 13/100 (13%) | 13/100 (13%) | 0 pp |
| nova | 9/100 (9%) | 8/100 (8%) | 11/100 (11%) | +2 pp |
| dw | 5/100 (5%) | 4/100 (4%) | 3/100 (3%) | −2 pp |
| **Overall** | **27/300 (9.0%)** | **25/300 (8.3%)** | **27/300 (9.0%)** | **0 pp** |

Compact metadata changed which questions the agent answered correctly, but not aggregate accuracy. The per-database differences are only a few questions and are descriptive, not evidence of improvement. The harness also has a fourth arm combining the full profile and metadata, but the headline run used three.

## How I tested

The implementation is in the [GitHub repository](https://github.com/renatyv/text2sql).

- Three MySQL dumps from BEAVER: `neutron`, `nova`, and `dw`.
- 100 seed-fixed questions per database per arm.
- pi coding agent with GPT-5.6 Luna PRO via OpenRouter at medium effort.
- A fresh Docker container per question, a `SELECT`-only database account, and network access limited to the database and OpenRouter.
- At most 15 turns per question.

I also found four harness problems—an implicit context limit, leaked `LIMIT 5`, truncated query output, and runs that ended without SQL. I describe the fixes in [Four Benchmark Settings That Can Distort a SQL Agent Test](/blog/text-to-sql-benchmark-harness/).

**Conclusion:** raw database access remained the simplest option and matched or beat both kinds of pre-generated context. Use a profile only when a different constraint—such as fewer live database calls—matters enough to justify it.
