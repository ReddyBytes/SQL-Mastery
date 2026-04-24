# SQL Mastery — Topic Recap

> One-line summary of every module. Use this to quickly review what each section covers before diving deeper.

---

## 01 — Fundamentals

| Topic | Summary |
|---|---|
| What is SQL | SQL as the universal language for relational databases — declarative, set-based queries |
| Databases & RDBMS | Tables, rows, columns, primary keys — the relational model |
| SQL vs NoSQL | When to choose relational vs document/key-value stores |
| Setup & Tools | Installing PostgreSQL/MySQL, psql, DBeaver, pgAdmin |

---

## 02 — Querying Basics

| Topic | Summary |
|---|---|
| SELECT | Retrieving columns, *, aliases — the foundation of every query |
| WHERE | Filtering rows with conditions, comparison operators, NULL handling |
| ORDER BY | Sorting results ascending/descending, multiple columns |
| DISTINCT | Removing duplicate rows from result sets |
| LIMIT & OFFSET | Pagination — fetching a subset of rows |

---

## 03 — Aggregation

| Topic | Summary |
|---|---|
| Aggregate Functions | COUNT, SUM, AVG, MIN, MAX — collapsing many rows to one value |
| GROUP BY | Grouping rows by column value before aggregation |
| HAVING | Filtering groups after aggregation (WHERE for aggregates) |
| Window Functions | ROW_NUMBER, RANK, LAG, LEAD — compute across rows without collapsing |
| ROLLUP & CUBE | Multi-level subtotals and cross-tabulations |

---

## 04 — Schema Design

| Topic | Summary |
|---|---|
| CREATE TABLE | Defining tables, columns, data types, constraints |
| Data Types | INT, VARCHAR, TEXT, BOOLEAN, DATE, NUMERIC — choosing the right type |
| Normalization | 1NF, 2NF, 3NF — eliminating redundancy and update anomalies |
| Indexes | B-tree, hash, partial indexes — speeding up reads at cost of write overhead |
| Constraints | PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, CHECK |

---

## 05 — Joins

| Topic | Summary |
|---|---|
| INNER JOIN | Return only rows with matching values in both tables |
| LEFT / RIGHT JOIN | Include all rows from one side, NULL where no match exists |
| FULL OUTER JOIN | All rows from both tables, NULL-filled where no match |
| CROSS JOIN | Cartesian product — every combination of rows |
| Self Join | Join a table to itself — hierarchies, comparisons within same table |
| Join Patterns | Multi-table joins, join ordering, join vs subquery trade-offs |

---

## 06 — Advanced Queries

| Topic | Summary |
|---|---|
| Subqueries | Nested SELECT — scalar, row, table subqueries, correlated subqueries |
| CTEs | WITH clause — named temporary result sets for readable complex queries |
| CASE | Conditional logic inside SELECT, dynamic column values |
| String Functions | UPPER, LOWER, TRIM, SUBSTRING, CONCAT, LIKE, REGEXP |
| Date Functions | NOW, EXTRACT, DATE_TRUNC, interval arithmetic |
| Set Operations | UNION, INTERSECT, EXCEPT — combining result sets |

---

## 07 — Data Modification

| Topic | Summary |
|---|---|
| INSERT | Adding rows — single row, multi-row, INSERT SELECT |
| UPDATE | Modifying existing rows — with conditions, from joins |
| DELETE | Removing rows — targeted deletion, TRUNCATE for full wipe |
| Transactions | BEGIN, COMMIT, ROLLBACK — grouping operations atomically |
| ACID Principles | Atomicity, Consistency, Isolation, Durability — guarantees of a transaction |
| Upsert | INSERT ON CONFLICT DO UPDATE — merge semantics |

---

## 08 — Performance

| Topic | Summary |
|---|---|
| Query Optimization | Rewriting queries to reduce rows scanned, push filters early |
| EXPLAIN ANALYZE | Reading execution plans — seq scan vs index scan, cost estimates |
| Index Strategy | When to index, composite indexes, covering indexes, index bloat |
| Partitioning | Range, list, hash partitioning for large tables |
| Connection Pooling | PgBouncer, connection limits, pool sizing |
| Query Patterns | N+1 problem, eager vs lazy loading, batch queries |

---

## 09 — Real World

| Topic | Summary |
|---|---|
| Views | Named saved queries — simplify complexity, provide access control |
| Stored Procedures | Reusable server-side logic in PL/pgSQL |
| Triggers | Auto-execute code on INSERT/UPDATE/DELETE events |
| SQL with Python | psycopg2, SQLAlchemy Core, SQLAlchemy ORM, Alembic migrations |
| Full-Text Search | tsvector, tsquery, GIN indexes — Postgres native search |

---

## 99 — Interview Master

| Topic | Summary |
|---|---|
| 26 Q&A Problems | Core SQL questions covering all levels from junior to senior |
| 25 Scenario Questions | Real-world design and debugging scenarios |
| Tricky Edge Cases | NULL behavior, aggregation gotchas, index pitfalls |

---

*Total modules: 9 + interview · Last updated: 2026-04-21*
