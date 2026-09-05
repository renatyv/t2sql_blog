---
title: "AI data analytic. Pitfalls in implementing AI agent"
description: "Four easy-to-miss settings that can make a text-to-SQL agent test misleading."
pubDate: "2026-08-24"
---

**TL;DR:** While building and AI data analytic angent I tried to limit him from spending too many tokens by modifying the harness. This backfired badly: resulting SQLs were not always correct.

## Introduction

Nowadays, almost every company wants an AI analytics feature. The idea is to have a UI where anyone can ask a question like, 'How much did we earn this week?' and get graphs and reports back.

Inevitably, one of the first things the AI agent behind the scenes does is generate SQL queries to extract valuable data from the database.

I did the same. But one cannot simply let the agent explore data freely. There are passwords and other sensitive information. Also, we can't allow it to spend $100 worth of tokens every time — we need to limit it somehow. My ideas to limit the AI agent looked reasonable:

- Ask the agent to add `LIMIT 5` to SQL queries it issues during exploration
- Limit the final result to 100 rows
- Limit the number of queries an agent can make during 'exploration' phase to 15

## Test setup

For the AI agent, I needed something lightweight and easy to modify, so I chose the popular [pi](https://pi.dev/) coding agent. It is as powerful as claude or codex, yet is easy to modify and get telemetry from: log tool calls, limit number of SQL queries an agent can submit to database, limit tool access.
It took only couple of prompts to limit number of 'turns', i.e. number of times he could send an SQL query and get back the result. In a similar way an 'sql_query' tool was created, which allowed to query the database without leaking login and password to the LLM. This tool also capped number of lines returned as a qery result to 100.

To test the resulting agent and harness I chose data from the [BEAVER](https://huggingface.co/datasets/BeaverBench/beaver) text-to-SQL benchmark. It contains database dumps and pairs of (text question, reference SQL query) – perfect for my needs. 

**Preventing AI from cheating.** The BEAVER benchmark is public, so anyone can see the correct 'gold' SQL queries for the questions in the dataset. To isolate the AI agent I used a fresh Docker container per question with a `SELECT`-only database account, and network access limited to the database and OpenRouter. This prevented AI from looking up answers locally, on the internet or leaking sensitive information from one question to another.

## Results and learnings

1. The agent sometimes copied the 'limit 5' into its final SQL even when the question asked for all matching rows. The query logic could be right while the answer remained incomplete. I removed it, and, yet had no problems with exceeding the token budget.
2. The database executed each query in full, but the tool showed the agent only the first 100 rows. This cap did not alter BEAVER's reference answer, and the evaluator still ran the final SQL separately. It restricted what the agent could inspect.
3. A **turn** is one agent response, which may contain database queries or final SQL. I allowed 15 turns per question to bound cost and runtime, but some runs spent all 15 inspecting tables and never answered. The fix was to force the agent on turn 13 to try to produce a final answer and use turn 14 to try to repair it using 'steering'. Turn 15 is now answer-only: database tools are disabled, and the agent must return its best SQL.
4. I tried the latest DeepSeek-V4-Flash, which was not recognized by PI agent at the time. As a result, the agent assumed that the context-window limit was 128k tokens. That was not enough for everything: the agent's prompt, the question, SQL queries, and query results.

## References

- [pi coding agent](https://pi.dev/)
- [BEAVER text-to-SQL benchmark dataset](https://huggingface.co/datasets/BeaverBench/beaver)
