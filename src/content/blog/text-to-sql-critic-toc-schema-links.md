---
title: "AI data analytic. Ideas for optimizing cost and speed"
description: "Reasoning effort, SQL critics, and profile retrieval tested on 498 BIRD Mini-Dev questions."
pubDate: "2026-08-26"
---

**TL;DR:** Low reasoning effort with raw database access is a good default for extracting data by generating SQL queries. It delivered the highest measured accuracy with the simplest setup on benchmark. If latency matters more, turn reasoning off and add the database profile bundle generated with [db-snooper](https://pypi.org/project/db-snooper/). Do not add a critic sub-agent: it cost ~50% more without improving accuracy.

I tested four ways on how to build AI data agent using [text-to-SQL benchmark](/blog/profiler-doesnt-help/): medium versus low reasoning, reasoning off, a fresh-context critic, and selective retrieval from a database profile. The low-effort and reasoning-off comparisons each used the same questions from [BIRD Mini-Dev](https://bird-bench.github.io/). The output for each question was an SQL query, which was executed and compared to reference (often called 'gold' query). Whether generated SQL matched the reference result - was the primary metric.

For an AI agent I chose [pi](https://pi.dev/), because it is a) open source b) minimal c) is made to be modified and adapted. [OpenRouter](https://openrouter.ai/) was chosen as LLM provider, because it allows to control how much money is spent and has good feedback info. See how agent was implemented in a separate [post](/blog/text-to-sql-benchmark-harness/). For the model I chose OpenAI GPT-5.6 Luna, because it is cheap, fast, and smart enough.

## 1. Low-effort raw access is the best default

At low effort, giving an AI agent read-only database access scored 311 out of 498 questions (62.4%). A [generated table summaries](https://pypi.org/project/db-snooper/)—table of contents, selected profile slices, and pre-generated [schema links](https://pypi.org/project/schema-linker/)—scored 306/498 (61.4%).

| Metric | Raw database access | Bundled table summary | Change |
|---|---:|---:|---:|
| Execution accuracy ↑ | 311/498 (62.4%) | 306/498 (61.4%) | <span class="metric-neutral">−1.0 pp</span> |
| DB queries/question ↓ | 4.50 | 1.55 | <span class="metric-good">−65.6%</span> |
| Latency/question ↓ | 27.8 s | 32.7 s | <span class="metric-bad">+17.7%</span> |
| Cost/question ↓ | $0.0074 | $0.0088 | <span class="metric-bad">+19.2%</span> |

The −1.0-point accuracy difference was not measurable (95% CI −3.3 to +1.3, exact McNemar p=.49). The profile won 14 of the 33 disagreements and lost 19. It reduced database traffic, but ran slower and cost more. The schema links generated with [schema-linker](https://pypi.org/project/schema-linker/) were bundled with the profile.

Low effort also dominated medium effort on cost and speed without a measurable accuracy loss:

| Metric | Low effort | Medium effort | Low-effort change |
|---|---:|---:|---:|
| Execution accuracy ↑ | 311/498 (62.4%) | 306/498 (61.4%) | <span class="metric-neutral">+1.0 pp</span> |
| Total tokens ↓ | 31.1M | 35.1M | <span class="metric-good">−11.3%</span> |
| Reasoning tokens ↓ | 0.59M | 1.04M | <span class="metric-good">−42.7%</span> |
| Total cost ↓ | $3.69 | $4.52 | <span class="metric-good">−18.3%</span> |
| Latency/question ↓ | 27.8 s | 35.9 s | <span class="metric-good">−22.7%</span> |
| DB queries/question ↓ | 4.50 | 5.90 | <span class="metric-good">−23.7%</span> |

## 2. Reasoning-off plus the profile is faster

Turning reasoning off made raw access much faster and cheaper than low effort, but reduced accuracy. Adding the profile bundle recovered most of the loss.

| Metric | Off, raw access | Off, bundled table summary | Low, raw access |
|---|---:|---:|---:|
| Execution accuracy ↑ | 287/498 (57.6%) | <span class="metric-neutral">303/498 (60.8%)</span> | 311/498 (62.4%) |
| Total tokens ↓ | 31.5M | <span class="metric-bad">43.3M</span> | 31.1M |
| Total cost ↓ | $2.96 | <span class="metric-neutral">$3.64</span> | $3.69 |
| Latency/question ↓ | 14.2 s | <span class="metric-good">14.8 s</span> | 27.8 s |
| DB queries/question ↓ | 5.36 | <span class="metric-good">1.57</span> | 4.50 |

The profile used 38% more tokens and cost 23% more than reasoning-off raw access, but made 71% fewer database queries with only 4% more latency. Compared with low-effort raw access, it was 1.6 points lower—a difference that was not measurable (95% CI −4.7 to +1.4, p=.37)—cost about the same, and ran 47% faster. The borderline p=.040 gain deserves replication.

## 3. Separate 'critic' AI agent did not help improving accueacy

Some papers and text-to-sql pipelines include so-called 'critic'. The critic read candidate SQL in a fresh context and checked projection, joins, filters, aggregation, and ordering. I tried it too, implemented as a subagent (just asked PI in the prompt). This resulted in no measurable accuracy improvement, but increased cost substantially.

| Metric | Inline self-critique | Critic sub-agent | Change |
|---|---:|---:|---:|
| Execution accuracy ↑ | 306/498 (61.4%) | 307/498 (61.6%) | <span class="metric-neutral">+0.2 pp</span> |
| Total tokens ↓ | 35.1M | 54.8M | <span class="metric-bad">+56%</span> |
| Total cost ↓ | $4.52 | $6.89 | <span class="metric-bad">+53%</span> |
| Latency/question ↓ | 35.9 s | 56.9 s | <span class="metric-bad">+58%</span> |


### Side quest: table of contents for table summary

The profiles generated by the [db-snoper](https://pypi.org/project/db-snooper/) can get large if there are many tables and columns. And the whole file is bundled into the prompt, eating precious tokens.

**Idea.** Generate a short table of contents for the generated tables summary, so that the agent can read summaries of only the table he needs to.

So I updated added the table of contents to db-snooper and re-run the same 500 queries.

| Signal | Full profile in prompt | TOC + selected slices | Change |
|---|---:|---:|---|
| Input-token premium over raw | about 40% | about 24% | <span class="metric-good">16 pp less overhead</span> |

Retrieval reduced prompt overhead, but the agent replaced SQL exploration with profile reads and did not become more accurate. The earlier full-profile run was faster, although that is not a clean speed comparison because the prompt and runner protocols changed.

## References

- [BIRD text-to-SQL benchmark](https://bird-bench.github.io/)
- [db-snooper](https://pypi.org/project/db-snooper/)
- [schema-linker](https://pypi.org/project/schema-linker/)
- [pi coding agent](https://pi.dev/)
- [OpenRouter](https://openrouter.ai/)
- [AI data analytic: All your agent needs is read-only database access](/blog/profiler-doesnt-help/)
- [AI data analytic: Pitfalls in implementing AI agent](/blog/text-to-sql-benchmark-harness/)
