---
title: "LLM-ready summary of your database"
description: "A compact database profile for SQL agents—and the mistakes that made it useful."
pubDate: "2026-08-27"
---

**TL;DR:** [db-snooper](https://github.com/renatyv/db-snooper) turns a database into compact Markdown for SQL agents. A useful profile is not a dump: it keeps the facts needed to understand tables and joins, makes them cheap to retrieve, and omits the rest.

I built db-snooper because a text-to-SQL agent kept relearning the same tables, joins, data types, and filter values. A plain schema was insufficient—`status text` says much less than `active=8,412, cancelled=327`—but sending every statistic created too much context.

The design question became: **how can a profile carry enough evidence without overwhelming the agent?** Three choices made it useful:

1. Keep schema and data shape together in one compact table block.
2. Let the agent retrieve only the blocks relevant to its question.
3. Bound expensive profiling and degrade gracefully when introspection fails.

## 1. Keep the useful facts together

db-snooper supports SQLite, PostgreSQL, MySQL, MariaDB, DuckDB, BigQuery, and matching RDS databases. It creates one Markdown profile per schema plus a table of contents.

Each table block combines structure with data shape:

```text
# "orders"  (rows=128420)

columns:
"id" bigint PK: unique identifier, 1..128420
"status" text: paid=91204, pending=21831, cancelled=15385
"customer_id" bigint FK: 18210 distinct, nulls=41
indexes: ("customer_id", "status")
```

Blocks can include types, constraints, distributions, indexes, relationships, and samples. Low-cardinality columns get histograms; small tables show every row. Empty and technical tables are skipped. Passwords, hashes, secrets, and tokens are redacted.

This makes the file useful for SQL generation, database exploration, migration review, and debugging without live production access. Because it contains real data, it must be protected like a database export.

## 2. Make the profile retrievable

The pipeline has three steps:

1. Inspect tables, views, columns, keys, and indexes.
2. Profile columns according to table size and data shape.
3. Render contiguous table blocks and record their line ranges in a table of contents.

For a large database, an agent reads the TOC and loads only the relevant blocks. One line per column and a hash-pinned TOC avoid repeating the same facts across DDL, statistics, and samples.

This matters because more context does not guarantee better SQL. In my first [benchmark](/blog/profiler-doesnt-help/), putting the full profile in every prompt used about six times more input tokens without improving accuracy. A later [retrieval experiment](/blog/text-to-sql-critic-toc-schema-links/) reduced that overhead by loading selected blocks.

## 3. Bound the work and handle imperfect databases

Profiling adapts to the data instead of applying one statistic everywhere:

- Identifiers get ranges rather than averages; enums get frequencies; high-cardinality text stays in samples.
- SQLite values are checked against declared types, so `numeric→text` and strings such as `00123` survive correctly.
- Table size and indexes determine whether counts, medians, distributions, content-shape checks, and JSON inspection run.
- Very large tables use PostgreSQL `pg_stats`, MySQL `COLUMN_STATISTICS`, or MariaDB `mysql.column_stats` instead of full scans. Estimates are marked with `≈` and `(from db stats)`.
- Queries have timeouts. BigQuery queries are dry-run against a byte budget. A failed metric does not abort the profile.
- If reflection fails on views, partial indexes, restricted accounts, or dialect plugins, db-snooper falls back to `pg_dump --schema-only` or `mysqldump --no-data` and skips data profiling rather than losing the table.

Delimited identifiers preserve spaces, reserved words, and case. One latest row plus two random rows gives samples some variety without turning the profile into a copy of the database.

**Conclusion:** generate database context once, but organize it for selective reading. The durable artifact is the smallest trustworthy map an agent can navigate—not the largest file the database can produce.

## References

- [db-snooper on GitHub](https://github.com/renatyv/db-snooper)
- [AI data analytic: All your agent needs is read-only database access](/blog/profiler-doesnt-help/)
- [AI data analytic: Ideas for optimizing cost and speed](/blog/text-to-sql-critic-toc-schema-links/)
