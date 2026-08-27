---
title: "What changed text-to-SQL accuracy—and what only changed cost"
description: "A SQL critic, profile retrieval, and reasoning-effort ablations on 498 BIRD Mini-Dev questions."
pubDate: "2026-08-26"
---

**Results** 
1) Low effort gives the same accuracy faster and is far cheaper than medium.
2) No thinking + db-snooper profile recovered 3.2 points and made 71% fewer database queries, while staying roughly twice as fast as low effort.
3) Critic sub-agent increased cost without improving accuracy. 

**Experiment:** The low-effort and reasoning-off runs each completed the same [BIRD Mini-Dev](https://bird-bench.github.io/) questions in both arms, without the critic. The primary metric is **execution accuracy**: the share of generated SQL queries whose result matches the gold query. Higher is better.

I analyzed four changes to the [text-to-SQL benchmark](/blog/profiler-doesnt-help/):

1. A sub-agent criticized every candidate query.
2. The database profile moved out of the prompt. The agent received a small table of contents and read only relevant profile ranges.
3. The main agent used medium reasoning effort instead of low.
4. The main agent turned reasoning off entirely.

The profile arm also received pre-generated schema links, but I did not test their effect separately.

## Results at a glance

| Metric | Raw database access | Profile + TOC + schema links | Change | What it means |
|---|---:|---:|---:|---|
| Execution accuracy ↑ | 311/498 (62.4%) | 306/498 (61.4%) | <span class="metric-neutral">−1.0 pp</span> | No measurable difference |
| DB queries/question ↓ | 4.50 | 1.55 | <span class="metric-good">−65.6%</span> | Fewer database calls |
| Latency/question ↓ | 27.8 s | 32.7 s | <span class="metric-bad">+17.7%</span> | Slower |
| Cost/question ↓ | $0.0074 | $0.0088 | <span class="metric-bad">+19.2%</span> | The profile bundle cost more |

The accuracy difference was **−1.0 percentage point** (95% CI −3.3 to +1.3, exact McNemar p=.49). The runs disagreed on 33 questions: the profile won 14 and lost 19.

<figure class="result-chart" role="img" aria-labelledby="accuracy-ci-caption">
  <svg viewBox="0 0 720 130" aria-hidden="true">
    <text x="360" y="20" text-anchor="middle" font-size="16">Accuracy difference, percentage points</text>
    <line x1="70" y1="65" x2="650" y2="65" stroke="currentColor" stroke-width="2" />
    <line x1="360" y1="35" x2="360" y2="88" stroke="currentColor" stroke-dasharray="5 5" opacity="0.45" />
    <line x1="124" y1="65" x2="451" y2="65" stroke="var(--accent)" stroke-width="6" />
    <line x1="124" y1="53" x2="124" y2="77" stroke="var(--accent)" stroke-width="3" />
    <line x1="451" y1="53" x2="451" y2="77" stroke="var(--accent)" stroke-width="3" />
    <circle cx="288" cy="65" r="8" fill="var(--accent)" />
    <text x="288" y="45" text-anchor="middle" font-size="15">−1.0 pp</text>
    <text x="70" y="105" text-anchor="middle" font-size="14">−4</text>
    <text x="360" y="105" text-anchor="middle" font-size="14">0</text>
    <text x="650" y="105" text-anchor="middle" font-size="14">+4</text>
  </svg>
  <figcaption id="accuracy-ci-caption">Paired accuracy difference with a 95% confidence interval. The interval crosses zero, so the experiment found no accuracy improvement.</figcaption>
</figure>

<p class="result-note"><strong>Bottom line:</strong> at low effort, the profile bundle reduced database traffic, but did not improve accuracy and made the run slower and more expensive.</p>

### Try low reasoning effort

I ran the same 498 raw-arm questions at low and medium effort, with inline self-critique and no critic sub-agent. Only the reasoning-effort setting changed.

| Metric | Low effort | Medium effort | Low-effort change | What it means |
|---|---:|---:|---:|---|
| Execution accuracy ↑ | 311/498 (62.4%) | 306/498 (61.4%) | <span class="metric-neutral">+1.0 pp</span> | No measurable difference |
| Total tokens ↓ | 31.1M | 35.1M | <span class="metric-good">−11.3%</span> | Fewer tokens |
| Reasoning tokens ↓ | 0.59M | 1.04M | <span class="metric-good">−42.7%</span> | Less hidden reasoning |
| Total cost ↓ | $3.69 | $4.52 | <span class="metric-good">−18.3%</span> | Cheaper |
| Latency/question ↓ | 27.8 s | 35.9 s | <span class="metric-good">−22.7%</span> | Faster |
| DB queries/question ↓ | 4.50 | 5.90 | <span class="metric-good">−23.7%</span> | Less exploration |

Low effort's **+1.0 percentage-point** accuracy difference was noise (95% CI −1.2 to +3.2, exact McNemar p=.47). It won 18 disagreements and lost 13. By difficulty, low was +2.0 points on simple questions, +0.8 on moderate questions, and tied on challenging questions; none of those differences was significant.

<p class="result-note"><strong>Verdict:</strong> medium reasoning bought no measurable accuracy and used more time, tokens, queries, and money. Low is the better default for this setup.</p>

### Turn reasoning off

I ran both arms on the same 498 questions with reasoning disabled, inline self-critique, and no critic sub-agent. Without the profile, turning reasoning off was much faster and cheaper than low effort, but less accurate. The profile recovered most of that loss.

| Metric | Off, raw access | Off, profile bundle | Low, raw access |
|---|---:|---:|---:|
| Execution accuracy ↑ | 287/498 (57.6%) | 303/498 (60.8%) | 311/498 (62.4%) |
| Total tokens ↓ | 31.5M | 43.3M | 31.1M |
| Total cost ↓ | $2.96 | $3.64 | $3.69 |
| Latency/question ↓ | 14.2 s | 14.8 s | 27.8 s |
| DB queries/question ↓ | 5.36 | 1.57 | 4.50 |

Raw access with reasoning off lost **4.8 percentage points** versus low effort (95% CI −7.8 to −1.9, exact McNemar p=.0018). Adding the profile then gained **3.2 points** over reasoning-off raw access (95% CI +0.3 to +6.1, p=.040). The off arms disagreed on 54 questions: the profile won 35 and lost 19.

At reasoning off, the profile used 38% more total tokens and cost 23% more than raw access, but made 71% fewer database queries with only 4% more latency. Its 60.8% accuracy was 1.6 points below low-effort raw access, a difference that was not measurable (95% CI −4.7 to +1.4, p=.37). It cost about the same and ran 47% faster.

<p class="result-note"><strong>Verdict:</strong> reasoning-off raw access is fast but less accurate. The profile recovered two-thirds of that loss and is the speed-first option, though the borderline p=.040 gain deserves replication. Low-effort raw access remains the highest-accuracy and simplest default.</p>

### Add a critic sub-agent

The critic reads the candidate SQL in a fresh context, compares it with the question, and looks for mistakes in projection, joins, filters, aggregation, and ordering.

To isolate the sub-agent, I reran the same raw-arm questions without it. The main agent still performed the same critique inline, so this tests fresh-context critic dispatch rather than critique versus no critique.

| Metric | Inline self-critique | Critic sub-agent | Change | What it means |
|---|---:|---:|---:|---|
| Execution accuracy ↑ | 306/498 (61.4%) | 307/498 (61.6%) | <span class="metric-neutral">+0.2 pp</span> | No measurable difference |
| Total tokens ↓ | 35.1M | 54.8M | <span class="metric-bad">+56%</span> | More tokens |
| Total cost ↓ | $4.52 | $6.89 | <span class="metric-bad">+53%</span> | More expensive |
| Latency/question ↓ | 35.9 s | 56.9 s | <span class="metric-bad">+58%</span> | Slower |

The paired accuracy difference was **+0.2 percentage points** (95% CI −2.1 to +2.5, exact McNemar p=1.00). Of the 33 questions where the runs disagreed, the critic arm won 17 and lost 16.

<p class="result-note"><strong>Verdict:</strong> no measurable accuracy gain for 56% more tokens, 53% more cost, and 58% more time.</p>

### Add a table of contents to the database profile

The profile contains descriptions and statistics for tables and columns. Instead of putting the whole file in the prompt, I gave the agent a small TOC and let it read only relevant line ranges.

| Signal | Full profile in prompt | TOC + selected slices | What changed |
|---|---:|---:|---|
| Input-token premium over raw | about 40% | about 24% | <span class="metric-good">16 pp less overhead</span> |
| TOC and slice reads | Not applicable | 498/498 runs (100%) | File reads replaced some SQL exploration |
| Accuracy win over raw | None | None | <span class="metric-neutral">The combined arm was −1.0 pp</span> |

The input-token premium shrank, but accuracy did not improve. The agent simply replaced SQL exploration with reading the profile file. The earlier full-profile run was faster, but this is not a clean speed ablation because the prompt and runner protocols changed between runs.

---

The earlier medium-effort profile arm stopped at 491 questions because I ran out of money. The low-effort and reasoning-off reruns completed all 498 available questions in both arms.
