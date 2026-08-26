---
title: "Three text-to-SQL ideas that did not move accuracy"
description: "A SQL critic, a profile table of contents, and schema links were useful in different ways—but did not beat raw database access."
pubDate: "2026-08-26"
---

**TL;DR:** The profile bundle made 59% fewer database queries, but accuracy stayed flat and cost rose 16%. The critic burned tokens, the table of contents replaced SQL exploration with file reads, and schema linking remains unproven.

**Experiment:** 500 [BIRD Mini-Dev](https://bird-bench.github.io/) questions were planned per arm. The raw arm completed 498; the profile arm completed 491. The primary metric is **execution accuracy**: the share of generated SQL queries whose result matches the gold query. Higher is better. The accuracy comparison below uses the 491 questions completed by both arms.

I changed the [text-to-SQL benchmark](/blog/profiler-doesnt-help/) in three ways:

1. A sub-agent criticized every candidate query.
2. The database profile moved out of the prompt. The agent received a small table of contents and read only relevant profile ranges.
3. The profile arm also received pre-generated schema links.

## Results at a glance

| Metric | Raw database access | Profile + TOC + schema links | Change | What it means |
|---|---:|---:|---:|---|
| Execution accuracy ↑ | 305/491 (62.1%) | 303/491 (61.7%) | −0.4 pp | No measurable difference |
| DB queries/question ↓ | 5.37 | 2.20 | −59.0% | Fewer database calls |
| Latency/question ↓ | 56.9 s | 56.0 s | −1.6% | Essentially unchanged |
| Cost/question ↓ | $0.0138 | $0.0160 | +15.9% | The profile bundle cost more |

The accuracy difference was **−0.4 percentage points** (95% CI −2.6 to +1.8, p=.86). Operational averages use all completed runs in each arm: 498 raw and 491 profile.

<figure class="result-chart" role="img" aria-labelledby="accuracy-ci-caption">
  <svg viewBox="0 0 720 130" aria-hidden="true">
    <text x="360" y="20" text-anchor="middle" font-size="16">Accuracy difference, percentage points</text>
    <line x1="70" y1="65" x2="650" y2="65" stroke="currentColor" stroke-width="2" />
    <line x1="360" y1="35" x2="360" y2="88" stroke="currentColor" stroke-dasharray="5 5" opacity="0.45" />
    <line x1="109" y1="65" x2="534" y2="65" stroke="var(--accent)" stroke-width="6" />
    <line x1="109" y1="53" x2="109" y2="77" stroke="var(--accent)" stroke-width="3" />
    <line x1="534" y1="53" x2="534" y2="77" stroke="var(--accent)" stroke-width="3" />
    <circle cx="321" cy="65" r="8" fill="var(--accent)" />
    <text x="321" y="45" text-anchor="middle" font-size="15">−0.4 pp</text>
    <text x="70" y="105" text-anchor="middle" font-size="14">−3</text>
    <text x="360" y="105" text-anchor="middle" font-size="14">0</text>
    <text x="650" y="105" text-anchor="middle" font-size="14">+3</text>
  </svg>
  <figcaption id="accuracy-ci-caption">Paired accuracy difference with a 95% confidence interval. The interval crosses zero, so the experiment found no accuracy improvement.</figcaption>
</figure>

<p class="result-note"><strong>Bottom line:</strong> the profile bundle reduced database traffic, but did not improve accuracy, latency, or cost.</p>

### 1. Add a critic sub-agent

It reads the candidate SQL in a fresh context, compares it with the question, and looks for mistakes in projection, joins, filters, aggregation, and ordering. It ran in 985/989 completed arm-runs (99.6%).

On the raw arm—the closest available comparison—the critic changed the result like this:

| Metric | Without critic | With critic | Change | What it means |
|---|---:|---:|---:|---|
| Execution accuracy ↑ | 303/498 (60.8%) | 307/498 (61.6%) | +0.8 pp | Statistical noise |
| Total tokens ↓ | 24.8M | 54.8M | +121% | More than doubled |
| Total cost ↓ | $4.01 | $6.89 | +72% | Much more expensive |

<p class="result-note"><strong>Verdict:</strong> no measurable accuracy gain for 121% more tokens.</p>

### 2. Add a table of contents to the database profile

The profile contains descriptions and statistics for tables and columns. Instead of putting the whole file in the prompt, I gave the agent a small TOC and let it read only relevant line ranges.

| Signal | Full profile in prompt | TOC + selected slices | What changed |
|---|---:|---:|---|
| Input-token premium over raw | about 40% | about 15% | 25 pp less overhead |
| TOC and slice reads | Not applicable | 491/491 runs (100%) | File reads replaced some SQL exploration |
| Accuracy win over raw | None | None | The combined arm was −0.4 pp |

The input-token premium shrank, but accuracy did not improve. The agent simply replaced SQL exploration with reading the profile file. For these small databases, putting the full profile in the prompt was at least faster. This is not a clean speed ablation—the new run also added the expensive critic and schema links.

### 3. Add schema links

| Evidence | Result | What it proves |
|---|---:|---|
| Runs that read schema links | 478/491 (97.4%) | The agent used the artifact |
| Isolated A/B comparison | Not run | Nothing about effectiveness |

<p class="result-note"><strong>Verdict: effect not measured separately.</strong> Usage is not evidence that schema links improved the answer.</p>

Schema links were bundled with the TOC and critic. There was no identical profile arm without links, and the combined arm did not beat raw access. The missing experiment is one boring A/B test: profile + TOC with links versus the same arm without links and without the critic.

---

The profile arm stopped at 491 rather than 500 questions because I ran out of money.
