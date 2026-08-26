---
title: "Three text-to-SQL ideas that did not move accuracy"
description: "A SQL critic, a profile table of contents, and schema links were useful in different ways—but did not beat raw database access."
pubDate: "2026-08-26"
---

**TL;DR:** The critic burns tokens, the table of contents just makes the agent read files instead of querying the database, and schema linking remains unproven.

I ran 500 [BIRD Mini-Dev](https://bird-bench.github.io/) questions after changing the [text-to-SQL benchmark](/blog/profiler-doesnt-help/) in three ways:

1. A sub-agent criticized every candidate query.
2. The database profile moved out of the prompt. The agent received a small table of contents and read only relevant profile ranges.
3. The profile arm also received pre-generated schema links.

| Arm | Execution accuracy | DB queries per question | Latency | Cost per question |
|---|---:|---:|---:|---:|
| Raw database access | 62.1% | 5.37 | 56.9s | $0.0138 |
| Profile + TOC + schema links | 61.7% | 2.20 | 56.0s | $0.0160 |

The difference was **−0.4 percentage points** (95% CI −2.6 to +1.8, p=.86). The profile bundle reduced database queries, but did not improve accuracy, latency, or cost.

### 1. Add a critic sub-agent

It reads the candidate SQL in a fresh context, compares it with the question, and looks for mistakes in projection, joins, filters, aggregation, and ordering.

**Result:** the critic ran in 985 of 989 completed arm-runs. On the raw arm—the closest available comparison—accuracy moved from 60.8% to 61.6%. That +0.8-point change was statistical noise, while total tokens increased by 121% and cost by 72%.

Useless shit that only burns tokens.

### 2. Add a table of contents to the database profile

The profile contains descriptions and statistics for tables and columns. Instead of putting the whole file in the prompt, I gave the agent a small TOC and let it read only relevant line ranges.

**Result:** every completed profile run read the TOC first and then requested profile slices. The profile arm's input-token premium over raw fell from about 40% to 15%.

Not one bit better than direct database access. The agent simply replaced SQL exploration with reading the profile file. For these small databases, putting the full profile in the prompt was at least faster. This is not a clean speed ablation—the new run also added the expensive critic—but there is no accuracy win here.

### 3. Add schema links

**Result:** unclear. Schema links were read in 478 of 491 profile runs, but they were bundled with the TOC and critic. There was no identical profile arm without links, and the combined arm did not beat raw access.

This experiment does not show that schema linking helps. It needs one boring A/B test: profile + TOC with links versus the same arm without links and without the critic.

---

It is 491 rather than 500 questions because I ran out of money.
