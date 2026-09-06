---
title: "Discovering Join Paths For Your AI Agent"
description: "A compact map of missing database joins for SQL agents—and the false positives that shaped it."
abstract: "Real databases often contain join paths that exist in the data but are absent from declared constraints, and schema-linker finds them. It treats every inferred relationship as a candidate: names propose it, several independent signals support it, and exact value containment verifies it."
pubDate: "2026-08-27"
---

The AI agent sometimes chooses the wrong tables or invents a join because real databases have incomplete foreign keys. For example, *'support_tickets.customer_id'* may refer to *'customers.customer_id'* without a declared constraint. The challenge is that shared names or values can also create convincing nonsense. I built [schema-linker](https://github.com/renatyv/schema-linker) tool to help an AI agent discover missing joins without mistaking coincidence for structure. It supports SQLite, PostgreSQL, MySQL, MariaDB, and DuckDB.

## How it works

schema-linker checks possible relationships from cheapest to most expensive, discarding weak candidates at each step.

### 1. Find plausible column pairs

It starts with declared primary and foreign keys, column names and types, row counts, and estimated distinct-value counts. Existing foreign keys are already known, incompatible data types cannot form useful joins, and near-unique free-text columns are poor identifiers, so those pairs are skipped. Similar names such as `orders.customer_id` and `customers.customer_id` identify the remaining pairs worth checking against the data.

### 2. Check which column points to which

A foreign-key relationship only needs to work in one direction: every value in `orders.customer_id` must exist in `customers.customer_id`, but many customers may have no orders. Suppose the orders contain customer IDs `{1, 2}` and the customers contain `{1, 2, 3, 4, 5}`. [Jaccard similarity](https://en.wikipedia.org/wiki/Jaccard_index) divides the two shared values by the five values found in either column, giving only 40%. schema-linker asks a more useful question: what percentage of the order customer IDs exist in the customer table? Here, both of them do, so the result is 100%. To avoid comparing every pair value by value, [MinHash](https://en.wikipedia.org/wiki/MinHash) creates a small fingerprint of each column's values, and [LSH Ensemble](https://en.wikipedia.org/wiki/Locality-sensitive_hashing) uses those fingerprints to find column pairs likely to match. Only those pairs receive an exact check: in memory for small sets, or with an SQL anti-join for large ones.

Finding every value from one column in the other is not enough by itself. Two unrelated flags containing `0` and `1` have the same values but would create a many-to-many join. schema-linker rejects such flag pairs unless one column is a primary or unique key, and reports a relationship only when at least three independent signals agree.

### 3. Group evidence into a compact join map

Pairwise output multiplies noise: four columns in one customer-ID domain can produce many redundant relationships. schema-linker groups them under a primary-key anchor. A report with declared links enabled can look like this:

```text
# Schema Links
- version: 0.0.5
- dialect: sqlite
- database: examples/shop.sqlite
- schema: main
## Declared PK/FK Links
order_lines.order_id -> orders.order_id
orders.customer_id -> customers.customer_id
## Inferred Links
### customers.customer_id
- inferred: support_tickets.customer_id
- declared: orders.customer_id
```

## References

1. [schema-linker on GitHub](https://github.com/renatyv/schema-linker)
1. [Automatic Metadata Extraction for Text-to-SQL](https://arxiv.org/abs/2505.19988)
1. [Jaccard index](https://en.wikipedia.org/wiki/Jaccard_index)
1. [MinHash](https://en.wikipedia.org/wiki/MinHash)
1. [Locality-sensitive hashing](https://en.wikipedia.org/wiki/Locality-sensitive_hashing)
