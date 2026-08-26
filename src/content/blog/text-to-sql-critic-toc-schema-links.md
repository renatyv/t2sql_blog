---
title: "Three text-to-SQL ideas that did not move accuracy"
description: "A SQL critic, a profile table of contents, and schema links were useful in different ways—but did not beat raw database access."
pubDate: "2026-08-26"
---

**TL;DR:** The critic cost too much, the table of contents is worse for small DBs, and schema linking remains unproven.

I ran 500 [BIRD Mini-Dev](https://bird-bench.github.io/) questions after changing the [text-to-SQL benchmark](/blog/profiler-doesnt-help/) in three ways:

1. A sub-agent criticized every candidate query.
2. The database profile moved out of the prompt. The agent received a small table of contents and read only relevant profile ranges.
3. The profile arm also received pre-generated schema links.

| Arm | Execution accuracy | DB queries per question | Latency | Cost per question |
|---|---:|---:|---:|---:|
| Raw database access | 62.1% | 5.37 | 56.9s | $0.0138 |
| Profile + TOC + schema links | 61.7% | 2.20 | 56.0s | $0.0160 |

The difference was **−0.4 percentage points** (95% CI −2.6 to +1.8, p=.86). The profile bundle reduced database queries, but did not improve accuracy, latency, or cost.

### The critic ran, but was useless

The critic was called in 985 of 989 completed arm-runs. On the raw arm—the closest available comparison—accuracy moved from 60.8% to 61.6%. That +0.8-point change was not significant, while total tokens increased by 121% and cost by 72%.

### The table of contents is useless for small databases.

Every completed profile run read the TOC first and then requested profile slices. The profile arm's input-token premium over raw fell from about 40% to 15%. But time-to-answer became the same as for agent with raw DB access.

### Schema linking is still unproven

Schema links were read in 478 of 491 profile runs bundled them with the TOC and critic. The combined arm did not beat raw access.

----
Its 491 and not 500 questions becuase I ran out of mony
