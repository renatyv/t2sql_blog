---
title: "A Database Profile Is a Retrieval Artifact, Not a Dump"
description: "A compact database profile for SQL agents—and the mistakes that made it useful."
pubDate: "2026-08-27"
---

**TL;DR:** [db-snooper](https://github.com/renatyv/db-snooper) turns a database into compact Markdown for SQL agents. The hard part was deciding what to omit.

## Why I created it

I needed text-to-SQL for an analytics page. Database access worked, but every run relearned the tables, joins, data types, and filter values.

The paper [Automatic Metadata Extraction for Text-to-SQL](https://arxiv.org/abs/2505.19988) inspired me to create db-snooper: generate database context once instead of making the model rediscover it for every question.

Models and agent harnesses will keep improving, so prompt tricks such as critics, rankers, and parallel candidate generation may be automated away or become irrelevant. A deterministic database profile should remain useful regardless of which model reads it.

A schema was not enough. `status text` says less than `active=8,412, cancelled=327`. Foreign keys show possible joins; samples and null rates show whether they are useful.

I wanted a reusable file for an agent to read it before writing or debugging SQL, without live production access. It also helps with database exploration and migration review. Because profiles contain real data, they must be protected like database exports.

More context does not guarantee better SQL. In my first [benchmark](/blog/profiler-doesnt-help/), including the full profile used about six times more input tokens without improving accuracy. Compact profiles and per-table retrieval worked better.

## What it does

db-snooper supports SQLite, PostgreSQL, MySQL, MariaDB, DuckDB, BigQuery, and matching RDS databases. It produces one Markdown profile per schema and a table of contents.

Each table keeps its schema and data shape together:

```text
# "orders"  (rows=128420)

columns:
"id" bigint PK: unique identifier, 1..128420
"status" text: paid=91204, pending=21831, cancelled=15385
"customer_id" bigint FK: 18210 distinct, nulls=41
indexes: ("customer_id", "status")
```

Blocks include types, constraints, distributions, indexes, relationships, and samples. Low-cardinality columns get histograms; small tables show every row. Empty and technical tables are skipped. Passwords, hashes, secrets, and tokens are redacted.

For a large database, an agent reads the TOC and loads only relevant table blocks. This reduced token overhead in a later [retrieval experiment](/blog/text-to-sql-critic-toc-schema-links/).

## How it works

The pipeline has three steps:

1. Inspect tables, views, columns, keys, and indexes.
2. Profile columns according to table size and data shape.
3. Render table blocks and record their line ranges in the TOC.

Small tables are printed in full. Ordinary tables get distributions and samples. Very large tables use catalog statistics instead of scans. Queries have timeouts; BigQuery queries are dry-run against a byte budget.

## Pitfalls that improved the profile

- **Too much metadata is less context.** Early versions repeated columns across DDL, statistics, and samples. One line per column, contiguous table blocks, and a hash-pinned TOC cut noise and enabled retrieval.
- **Declared types and names lie.** SQLite values can contradict their declared type, so db-snooper reports `numeric→text` and keeps `00123` as text. Delimited identifiers preserve spaces, reserved words, and case.
- **Generic statistics are noise.** Identifiers get ranges instead of averages, enums get frequencies, small tables show every row, and high-cardinality text leaves the samples. One latest row and two random rows provide enough variety.
- **Profiling needs guards.** Table size and indexes decide whether to run counts, medians, distributions, content-shape checks, and JSON inspection. JSON work is bounded by row count, sample count, and value size. Queries have timeouts, BigQuery has a scan budget, and a failed metric does not abort the profile.
- **Large tables must not be scanned.** At 100 million estimated rows, db-snooper skips `COUNT(*)`, samples, and column queries. It uses PostgreSQL `pg_stats`, MySQL `COLUMN_STATISTICS`, or MariaDB `mysql.column_stats`, marking estimates with `≈` and `(from db stats)`.
- **Introspection fails.** Views, partial indexes, restricted accounts, and dialect plugins expose different metadata. If reflection fails, db-snooper falls back to `pg_dump --schema-only` or `mysqldump --no-data` for DDL and skips data profiling instead of losing the table or crashing.

A database profile is a retrieval artifact, not a dump. Keep useful facts, make them cheap to find, and omit everything else.
