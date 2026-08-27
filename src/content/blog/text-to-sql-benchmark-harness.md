---
title: "Your text-to-SQL benchmark is testing the harness"
description: "Four small harness decisions that can make a working text-to-SQL agent look broken."
pubDate: "2026-08-24"
---

**Results**
1) Asking for `LIMIT 5` during exploration leaked the limit into final SQL.
2) Capping database responses at 100 rows made correct queries fail evaluation.
3) Letting the agent use all 15 turns for exploration produced missing answers.
4) Relying on the model's default context window made runs unpredictable.

**Experiment:** While testing whether a [database profile helps a coding agent](/blog/profiler-doesnt-help/), I observed four harness failure modes. I did not isolate a separate accuracy effect for each one. The values `5`, `100`, `15`, and `128k` below are configuration values, not accuracy results.

## Results at a glance

| Harness decision | What broke | Score symptom | Final rule |
|---|---|---|---|
| Ask for `LIMIT 5` during exploration | The limit leaked into final SQL | Correct logic, incomplete result set | Do not put the limit instruction in the prompt |
| Return at most 100 rows | Some gold queries return thousands | Correct SQL scored against truncated data | Return complete query results |
| Allow all 15 turns for exploration | Some runs ended without SQL | Missing answer counted as failure | Reserve the last turn for SQL |
| Rely on the model's context default | pi used 128k for unknown models | Unexpected truncation or spend | Set the context window explicitly |

<p class="result-note"><strong>Important:</strong> the table explains failure mechanisms, not effect sizes. The benchmark recorded the failures, but these four changes were not isolated in separate A/B runs.</p>

### 1. LIMIT 5 leaked to the answer

I told the agent to add `LIMIT 5` to exploratory queries. The `5` was meant to keep inspection output short. The agent then appended `LIMIT 5` to some final queries, changing the result set used for execution accuracy.

### 2. A row cap changed the expected result

I also capped database responses at 100 rows. The `100` was a transport cap, not a property of the benchmark. It made runs cheaper, but some BEAVER gold queries return thousands of rows.

### 3. The agent used every turn without returning SQL

An agent spent its full 15-turn budget inspecting tables. I reserved turn 15 for the answer and added a reminder on turn 14. The evaluator still records missing answers as failures, but the harness now gives the agent a predictable chance to finish.

### 4. The context window was not the one I expected

pi defaults to a 128k-token context window for unknown models. That number is a fallback setting, not the model limit I intended to test.

### The boring harness I ended up with

- A fresh Docker container per question
- A SELECT-only database account
- The same seed-fixed questions for every arm
- Complete query results
- At most 15 turns, with the final turn reserved for SQL
- An explicitly configured context window
- Token, turn, and database-query counts recorded alongside accuracy
