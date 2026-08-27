---
title: "Why I built database schema-linker"
description: "A compact map of missing database joins for SQL agents—and the false positives that shaped it."
pubDate: "2026-08-27"
---

**TL;DR:** [schema-linker](https://github.com/renatyv/schema-linker) finds relationships that exist in the data but are missing from the database constraints. It writes a compact Markdown map for SQL agents. The hard part was rejecting joins that looked convincing but were wrong.

## Why I created it

Text-to-SQL often fails before the SQL becomes complicated: it chooses the wrong tables or invents a join. Foreign keys help, but real databases have incomplete constraints. `support_tickets.customer_id` may refer to `customers.customer_id` without the database saying so.

The project was inspired by the paper [*Automatic Metadata Extraction for Text-to-SQL*](https://arxiv.org/abs/2505.19988). Its central idea—that understanding database contents is often harder than writing the query—led me to treat missing join paths as metadata that could be extracted automatically.

I wanted to discover these missing relationships once and give them to an agent as reusable context. The same file helps with multi-table queries, unfamiliar databases, join debugging, and legacy-schema documentation.

## What it does

schema-linker supports SQLite, PostgreSQL, MySQL, MariaDB, and DuckDB. It reads declared keys, infers additional join candidates, and groups columns that share a value domain:

```text
### customers.customer_id
- inferred: support_tickets.customer_id
- declared: orders.customer_id
```

The declared link gives context; the inferred link is the new signal. It remains a candidate because shared values do not prove shared business meaning. Detailed evidence and declared-link sections are available when debugging but omitted by default to save tokens.

## How it works

The pipeline starts cheaply and spends more work only on plausible columns:

1. Read tables, keys, types, row counts, and distinct counts.
2. Drop known foreign keys, incompatible types, empty columns, and near-unique text.
3. Generate candidates from names, ID shape, cardinality, and MinHash containment.
4. Verify survivors with exact directional containment and require several signals.

Containment is directional. Every `orders.customer_id` can appear in `customers.customer_id` even when many customers have no orders. A symmetric similarity score can make that valid relationship look weak.

Large value sets are verified in SQL with an anti-join instead of being loaded into memory. Query timeouts and table-size gates keep discovery bounded.

## Pitfalls that improved the output

- **Matching names are not relationships.** `status`, `type`, and even `customer_id` can belong to unrelated domains. Names now nominate candidates; values, types, cardinality, and direction must support them.
- **Jaccard was the wrong comparison.** Parent and child columns often have very different set sizes. Directional containment finds these joins without requiring both sets to be equally complete.
- **Small domains create perfect nonsense.** Two independent flags both containing `0` and `1` have perfect containment but produce a cross-product when joined. Flag pairs are rejected unless one side is a primary or unique key.
- **Approximation is not verification.** MinHash and LSH Ensemble cheaply find candidates, but every reported link must pass an exact containment check in memory or in SQL.
- **Pairwise output multiplies noise.** Four columns in one customer-ID domain can produce many redundant pairs. Grouping them under one primary-key anchor turns the pairwise explosion into one readable block.
- **The output must be valid SQL context.** Dialect-aware quoting preserves spaces, reserved words, and case. Declared links and evidence stay optional so the default file contains only the new signal.

A schema-link file is a map of likely join paths, not proof. Combine weak signals, verify them against real values, and keep the output smaller than the problem it explains.
