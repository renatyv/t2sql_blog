---
title: "Your Database Is Hiding Join Paths From Your SQL Agent"
description: "A compact map of missing database joins for SQL agents—and the false positives that shaped it."
pubDate: "2026-08-27"
---

**TL;DR:** [schema-linker](https://github.com/renatyv/schema-linker) finds join paths that exist in the data but are absent from database constraints. It treats every inferred relationship as a candidate: names propose it, several independent signals support it, and exact value containment verifies it.

Text-to-SQL often fails before the SQL becomes complicated. The agent chooses the wrong tables or invents a join because real databases have incomplete foreign keys. For example, `support_tickets.customer_id` may refer to `customers.customer_id` without a declared constraint.

The challenge is that shared names or values can also create convincing nonsense. I built schema-linker to answer one question: **how can an agent discover missing joins without mistaking coincidence for structure?** The answer has three parts:

1. Narrow candidates with cheap structural signals.
2. Verify direction against actual values.
3. Present related columns as one compact domain, not a noisy list of pairs.

## 1. Use names to nominate, not prove

The project was inspired by [*Automatic Metadata Extraction for Text-to-SQL*](https://arxiv.org/abs/2505.19988): understanding database contents is often harder than writing the query.

schema-linker supports SQLite, PostgreSQL, MySQL, MariaDB, and DuckDB. It begins with tables, keys, types, row counts, and distinct counts, then rejects known foreign keys, incompatible types, empty columns, and near-unique text.

Names, ID shape, cardinality, and MinHash containment nominate the remaining candidates. No single signal is enough. Columns called `status`, `type`, or even `customer_id` can belong to unrelated domains.

## 2. Verify directional containment exactly

Parent and child columns often have very different set sizes. Every `orders.customer_id` may appear in `customers.customer_id` even when many customers have no orders, so symmetric Jaccard similarity can make a valid relationship look weak.

schema-linker instead checks directional containment. MinHash and LSH Ensemble reduce the candidate set; every reported relationship must then pass an exact containment check in memory or SQL. Large sets use an anti-join rather than being loaded into memory.

Small domains need an extra guard. Two unrelated flags containing `0` and `1` have perfect containment but produce a cross-product when joined. Flag pairs are rejected unless one side is a primary or unique key.

## 3. Group evidence into a compact join map

Pairwise output multiplies noise: four columns in one customer-ID domain can produce many redundant relationships. schema-linker groups them under a primary-key anchor:

```text
### customers.customer_id
- inferred: support_tickets.customer_id
- declared: orders.customer_id
```

The declared relationship supplies context; the inferred relationship is the new signal. Detailed evidence and declared-only sections remain available for debugging but are omitted by default to save tokens. Dialect-aware quoting preserves spaces, reserved words, and case.

The same map helps with multi-table SQL, unfamiliar databases, join debugging, and legacy-schema documentation. Query timeouts and table-size gates keep discovery bounded.

**Conclusion:** a schema-link file is evidence about likely join paths, not proof of business meaning. Combine weak signals, verify them against real values, and keep the result smaller than the problem it explains.
