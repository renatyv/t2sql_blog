---
title: "Your text-to-SQL benchmark is testing the harness"
description: "Four small harness decisions that can make a working text-to-SQL agent look broken."
pubDate: "2026-08-24"
---

**TL;DR:** Simple ways to make text2SQL worse.

While testing whether a [database profile helps a coding agent](/blog/profiler-doesnt-help/), I found several ways to ruin the benchmark results

### 1. LIMIT 5 leaked to the answer

I told the agent to add `LIMIT 5` to exploratory queries. The agent then appended `LIMIT 5` to some final queries.

### 2. A row cap changed the expected result

I also capped database responses at 100 rows. That made runs cheaper, but some BEAVER gold queries return thousands of rows.

### 3. The agent used every turn without returning SQL

An agent spent its full turns budget inspecting tables. I reserved the last turn for the answer and added a reminder on the penultimate turn. The evaluator still records missing answers as failures, but the harness now gives the agent a predictable chance to finish.

### 4. The context window was not the one I expected

pi defaults to a 128k context window for unknown models.

## The boring harness I ended up with

- A fresh Docker container per question
- A SELECT-only database account
- The same seed-fixed questions for every arm
- Complete query results
- At most 15 turns, with the final turn reserved for SQL
- Token, turn, and database-query counts recorded alongside accuracy
