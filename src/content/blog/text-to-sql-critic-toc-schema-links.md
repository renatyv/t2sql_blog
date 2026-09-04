---
title: "Use Low Reasoning for SQL Accuracy—and Skip the Critic"
description: "Reasoning effort, SQL critics, and profile retrieval tested on 498 BIRD Mini-Dev questions."
pubDate: "2026-08-26"
---

**TL;DR:** Use low reasoning effort with raw database access as the default for this SQL agent. It delivered the highest measured accuracy with the simplest setup. If latency matters more, turn reasoning off and add the profile bundle. Do not add a critic sub-agent: it cost 53% more without improving accuracy.

I tested four changes to the earlier [text-to-SQL benchmark](/blog/profiler-doesnt-help/): medium versus low reasoning, reasoning off, a fresh-context critic, and selective retrieval from a database profile. The low-effort and reasoning-off comparisons each used the same 498 [BIRD Mini-Dev](https://bird-bench.github.io/) questions in both arms. **Execution accuracy**—whether generated SQL matched the reference result—was the primary metric.

The practical decision has three parts:

1. Choose low-effort raw access when accuracy is the priority.
2. Choose reasoning-off plus the profile bundle when speed is the priority.
3. Skip the critic and do not expect profile retrieval alone to improve accuracy.

## 1. Low-effort raw access is the best default

At low effort, raw database access scored 311/498 (62.4%). A profile bundle—table of contents, selected profile slices, and pre-generated schema links—scored 306/498 (61.4%).

| Metric | Raw database access | Profile bundle | Change |
|---|---:|---:|---:|
| Execution accuracy ↑ | 311/498 (62.4%) | 306/498 (61.4%) | <span class="metric-neutral">−1.0 pp</span> |
| DB queries/question ↓ | 4.50 | 1.55 | <span class="metric-good">−65.6%</span> |
| Latency/question ↓ | 27.8 s | 32.7 s | <span class="metric-bad">+17.7%</span> |
| Cost/question ↓ | $0.0074 | $0.0088 | <span class="metric-bad">+19.2%</span> |

The −1.0-point accuracy difference was not measurable (95% CI −3.3 to +1.3, exact McNemar p=.49). The profile won 14 of the 33 disagreements and lost 19. It reduced database traffic, but ran slower and cost more. Because schema links were bundled with the profile, this experiment does not isolate their effect.

Low effort also dominated medium effort on cost and speed without a measurable accuracy loss:

| Metric | Low effort | Medium effort | Low-effort change |
|---|---:|---:|---:|
| Execution accuracy ↑ | 311/498 (62.4%) | 306/498 (61.4%) | <span class="metric-neutral">+1.0 pp</span> |
| Total tokens ↓ | 31.1M | 35.1M | <span class="metric-good">−11.3%</span> |
| Reasoning tokens ↓ | 0.59M | 1.04M | <span class="metric-good">−42.7%</span> |
| Total cost ↓ | $3.69 | $4.52 | <span class="metric-good">−18.3%</span> |
| Latency/question ↓ | 27.8 s | 35.9 s | <span class="metric-good">−22.7%</span> |
| DB queries/question ↓ | 4.50 | 5.90 | <span class="metric-good">−23.7%</span> |

Only the reasoning setting changed: both arms used raw access, inline self-critique, and no critic sub-agent. Low effort's +1.0-point difference was noise (95% CI −1.2 to +3.2, exact McNemar p=.47). It won 18 disagreements and lost 13. Differences by difficulty—+2.0 points on simple questions, +0.8 on moderate questions, and zero on challenging questions—were also not significant.

## 2. Reasoning-off plus the profile is the speed-first option

Turning reasoning off made raw access much faster and cheaper than low effort, but reduced accuracy. Adding the profile bundle recovered most of the loss.

| Metric | Off, raw access | Off, profile bundle | Low, raw access |
|---|---:|---:|---:|
| Execution accuracy ↑ | 287/498 (57.6%) | 303/498 (60.8%) | 311/498 (62.4%) |
| Total tokens ↓ | 31.5M | 43.3M | 31.1M |
| Total cost ↓ | $2.96 | $3.64 | $3.69 |
| Latency/question ↓ | 14.2 s | 14.8 s | 27.8 s |
| DB queries/question ↓ | 5.36 | 1.57 | 4.50 |

Reasoning-off raw access lost 4.8 points versus low effort (95% CI −7.8 to −1.9, exact McNemar p=.0018). The profile then added 3.2 points over reasoning-off raw access (95% CI +0.3 to +6.1, p=.040), winning 35 of 54 disagreements and losing 19.

The profile used 38% more tokens and cost 23% more than reasoning-off raw access, but made 71% fewer database queries with only 4% more latency. Compared with low-effort raw access, it was 1.6 points lower—a difference that was not measurable (95% CI −4.7 to +1.4, p=.37)—cost about the same, and ran 47% faster. The borderline p=.040 gain deserves replication.

## 3. The extra machinery did not improve accuracy

### A critic added cost, not correctness

The critic read candidate SQL in a fresh context and checked projection, joins, filters, aggregation, and ordering. To isolate fresh-context dispatch, the comparison arm performed the same critique inline.

| Metric | Inline self-critique | Critic sub-agent | Change |
|---|---:|---:|---:|
| Execution accuracy ↑ | 306/498 (61.4%) | 307/498 (61.6%) | <span class="metric-neutral">+0.2 pp</span> |
| Total tokens ↓ | 35.1M | 54.8M | <span class="metric-bad">+56%</span> |
| Total cost ↓ | $4.52 | $6.89 | <span class="metric-bad">+53%</span> |
| Latency/question ↓ | 35.9 s | 56.9 s | <span class="metric-bad">+58%</span> |

The +0.2-point difference was not measurable (95% CI −2.1 to +2.5, exact McNemar p=1.00). The critic won 17 of 33 disagreements and lost 16.

### Selective profile retrieval saved context, not accuracy

Instead of putting the whole profile in the prompt, I gave the agent a table of contents and let it read relevant line ranges.

| Signal | Full profile in prompt | TOC + selected slices | Change |
|---|---:|---:|---|
| Input-token premium over raw | about 40% | about 24% | <span class="metric-good">16 pp less overhead</span> |
| TOC and slice reads | Not applicable | 498/498 runs (100%) | File reads replaced some SQL exploration |
| Accuracy win over raw | None | None | <span class="metric-neutral">Combined arm was −1.0 pp</span> |

Retrieval reduced prompt overhead, but the agent replaced SQL exploration with profile reads and did not become more accurate. The earlier full-profile run was faster, although that is not a clean speed comparison because the prompt and runner protocols changed.

The earlier medium-effort profile arm stopped at 491 questions when I ran out of money. Both arms completed all 498 questions in the low-effort and reasoning-off reruns.

**Conclusion:** low-effort raw access is the accuracy-first default. Reasoning-off plus the profile is a credible speed-first alternative. The critic and selective profile retrieval add complexity without an accuracy result to justify it.
