---
title: "How I ruined my AI data pipeline by overengineering it"
description: "Four easy-to-miss settings that can make a text-to-SQL agent test misleading."
pubDate: "2026-08-24"
---

**TL;DR:** An AI agent trying to use the SQL database is credible only if the harness lets the agent return complete SQL under explicit, reproducible limits. I found four settings that violated that rule: a leaked `LIMIT 5`, a 100-row display cap, no reserved answer turn, and an implicit 128k context window.

I found these problems while testing whether a [database profile helps a coding agent](/blog/profiler-doesnt-help/). I used [BEAVER](https://huggingface.co/datasets/BeaverBench/beaver), where every question has a database and a reference SQL query, and popular [pi](https://pi.dev/) coding agent that could inspect the database and return SQL.

The goal was to measure SQL ability. The complication was that small runner conveniences could change what the agent wrote, hide data from it, or prevent an answer. I therefore made four controls explicit:

| Test setting | Failure | Control |
|---|---|---|
| Ask for `LIMIT 5` during exploration | The limit leaked into final SQL | Let the agent choose when to sample |
| Show at most 100 result rows | The agent could not inspect a complete large result | Return complete results |
| Allow 15 responses without reserving an answer | A run could end before producing SQL | Make response 15 answer-only |
| Let pi choose the context window | It used a 128,000-token fallback | Set the value explicitly |

The values `5`, `100`, `15`, and `128k` are configuration settings, not accuracy results. I observed these failure modes while building the runner; I did not isolate an accuracy effect for each fix.

## 1. Keep exploration hints out of the final answer

I asked the agent to add `LIMIT 5` to exploratory queries to keep output short. It sometimes copied the limit into its final SQL even when the question asked for all matching rows. The query logic could be right while the answer remained incomplete.

I removed the instruction. The agent can still add a small limit when it needs a sample.

## 2. Let the agent inspect complete results

The database executed each query in full, but the tool showed the agent only the first 100 rows. Some BEAVER reference queries return thousands.

This cap did not alter BEAVER's reference answer, and the evaluator still ran the final SQL separately. It restricted what the agent could inspect. I removed the cap; exploratory queries can still use an intentional `LIMIT`.

## 3. Reserve a turn for the answer

A **turn** is one agent response, which may contain database queries or final SQL. I allowed 15 turns per question to bound cost and runtime, but one run spent all 15 inspecting tables and never answered.

Turn 15 is now answer-only: database tools are disabled after turn 14, and the agent must return its best SQL. Missing SQL still counts as failure.

## 4. Set the context window explicitly

The **context window** covers the question, instructions, conversation, and database output. For an unrecognized model, pi assumed 128,000 tokens. That was a fallback, not a deliberate experiment setting or necessarily the model's actual limit.

I now configure the window explicitly so repeated runs use the same intended limit.
