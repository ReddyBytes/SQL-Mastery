# Indexes

## The Book's Index

At the back of most textbooks there is an index: an alphabetical list of topics with the page numbers where they appear. If you want to find everything about "photosynthesis" in a 600-page biology book, you do not read from page 1. You flip to the index, find "photosynthesis → pages 82, 243, 391," and go straight there.

A database index works identically. Without an index, the database reads every row in the table from top to bottom looking for matches — a **full table scan**. With an index on the right column, it jumps directly to the matching rows in a fraction of the time.

```
Without index:                    With index on email:
  Read row 1  ... no match          Look up 'alice@x.com' in index
  Read row 2  ... no match          → points to row 41,823
  Read row 3  ... no match          Fetch row 41,823 directly
  ...                               Done. (1 read instead of 100,000)
  Read row 41,823 ... MATCH!
  ...
  Read row 100,000 ... done
```

---

## How a B-Tree Index Works (Simplified)

Most database indexes use a **B-tree (Balanced Tree)** structure. It keeps values sorted, making range queries and equality lookups fast.

```
B-tree index on users.email:

                        [ M ]
                       /     \
              [ D | H ]       [ R | W ]
             /    |    \       /    |   \
         [A-C] [E-G] [I-L] [N-Q] [S-V] [X-Z]
          |                  |
    alice@x.com         nancy@x.com
    bob@x.com           oliver@x.com
    carol@x.com         ...

Each leaf node contains the index value + a pointer to the actual row on disk.
Searching for "nancy@x.com": root -> right subtree -> [N-Q] leaf -> row pointer -> fetch row.
The tree stays balanced — all leaves are the same depth.
```

B-trees are efficient for:
- Equality: `WHERE email = 'alice@example.com'`
- Ranges: `WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'`
- Sorting: `ORDER BY last_name`
- Prefix matching: `WHERE name LIKE 'Smith%'`

They are **not** efficient for:
- Suffix/infix patterns: `WHERE name LIKE '%Smith'` (cannot use the sorted structure)
- Very low-cardinality columns: `WHERE is_active = TRUE` (index on a boolean is almost never useful)

---

## Creating Indexes

### Basic Index

```sql
-- Create an index on the email column
CREATE INDEX idx_users_email ON users(email);

-- Naming convention: idx_tablename_columnname
```

### UNIQUE Index

Enforces uniqueness AND speeds up lookups. When you add a `UNIQUE` constraint to a column, PostgreSQL automatically creates a unique index.

```sql
-- Explicit unique index
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- This is equivalent to the constraint:
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);
```

### Composite (Multi-Column) Index

An index on multiple columns — useful when you frequently filter or sort by a combination.

```sql
-- Orders often queried by customer AND status together
CREATE INDEX idx_orders_customer_status
    ON orders(customer_id, status);
```

**Column order matters.** This index is useful for:
- `WHERE customer_id = 101`
- `WHERE customer_id = 101 AND status = 'shipped'`

But it is **not** useful for:
- `WHERE status = 'shipped'` alone (the first column is not in the filter)

This is called the **leftmost prefix rule**: a composite index `(a, b, c)` can serve queries on `(a)`, `(a, b)`, or `(a, b, c)` — but not `(b)`, `(c)`, or `(b, c)` alone.

### Partial Index

Index only a subset of rows — dramatically smaller and faster when most queries target a specific condition.

```sql
-- Only index active users (ignores the majority who are inactive)
CREATE INDEX idx_users_active_email
    ON users(email)
    WHERE is_active = TRUE;

-- Only index unpaid orders (once paid, they are rarely queried)
CREATE INDEX idx_orders_pending
    ON orders(created_at)
    WHERE status = 'pending';
```

---

## EXPLAIN — Seeing the Index in Action

PostgreSQL's `EXPLAIN` and `EXPLAIN ANALYZE` show exactly how the query planner will execute a query. This is how you confirm whether an index is being used.

### Without an Index

```sql
-- No index on orders.customer_id yet
EXPLAIN SELECT * FROM orders WHERE customer_id = 101;
```

```
Seq Scan on orders  (cost=0.00..2834.00 rows=47 width=72)
  Filter: (customer_id = 101)
```

`Seq Scan` = sequential scan = reading every row. Cost `2834` is proportional to the number of rows in the table.

### After Creating an Index

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);

EXPLAIN SELECT * FROM orders WHERE customer_id = 101;
```

```
Index Scan using idx_orders_customer_id on orders  (cost=0.43..98.12 rows=47 width=72)
  Index Cond: (customer_id = 101)
```

`Index Scan` = the index was used. Cost dropped from `2834` to `98`. The database went straight to the matching rows.

### EXPLAIN ANALYZE — Actual Runtime

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 101;
```

```
Index Scan using idx_orders_customer_id on orders
  (cost=0.43..98.12 rows=47 width=72)
  (actual time=0.042..0.831 rows=47 loops=1)
Planning Time: 0.189 ms
Execution Time: 0.902 ms
```

`EXPLAIN` shows the planner's estimate. `EXPLAIN ANALYZE` runs the query and shows actual times. Always use `EXPLAIN ANALYZE` when diagnosing real performance issues.

---

## When Indexes Help

```
Use an index when...
  - The column appears frequently in WHERE clauses
  - The column is used in JOIN conditions
  - The column is sorted with ORDER BY on large result sets
  - High cardinality: many distinct values (user IDs, emails, UUIDs)
  - You query a small percentage of rows (< ~5-10% of the table)

Examples of good index candidates:
  users(email)          -- unique, high cardinality, frequent lookups
  orders(customer_id)   -- FK column, frequently joined
  orders(status, created_at) -- frequently filtered + sorted together
  products(category_id) -- FK column, frequently filtered
```

---

## When Indexes Hurt

Indexes are not free. Every index adds overhead to `INSERT`, `UPDATE`, and `DELETE` because the index structure must be kept in sync with the table data.

```
Avoid an index when...
  - The table is small (< a few thousand rows)
    A full scan on a small table is faster than the index overhead.

  - Low cardinality columns: boolean, status with 3 values, gender
    WHERE is_active = TRUE might match 95% of rows — the index does not help.

  - Write-heavy tables with infrequent reads
    Bulk import tables, raw event logs being inserted at high rate.

  - Columns never used in WHERE / JOIN / ORDER BY
    Indexing them wastes storage and write performance.
```

---

## Dropping and Listing Indexes

```sql
-- Drop an index
DROP INDEX idx_users_email;

-- Drop safely (no error if it does not exist)
DROP INDEX IF EXISTS idx_users_email;

-- List all indexes on a table (PostgreSQL)
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders';
```

---

## Index Types Beyond B-Tree (PostgreSQL)

```
+----------+------------------------------------------------------------+
| Type     | Best for                                                   |
+----------+------------------------------------------------------------+
| B-tree   | Equality, ranges, sorting. The default. Use for most things|
| Hash     | Equality only (= operator). Rarely faster than B-tree.     |
| GIN      | Full-text search, JSONB keys, arrays                       |
| GiST     | Geometric data, range types, full-text (tsvector)          |
| BRIN     | Very large append-only tables sorted by a natural order    |
|          | (e.g. a time-series table partitioned by date)             |
+----------+------------------------------------------------------------+
```

For most application developers, B-tree (the default) is all you need. GIN becomes relevant when working with full-text search or JSONB queries.

---

## Summary

```
+---------------------+------------------------------------------------------+
| Command             | What it does                                         |
+---------------------+------------------------------------------------------+
| CREATE INDEX        | Build a B-tree index on one or more columns          |
| CREATE UNIQUE INDEX | Build an index that enforces uniqueness              |
| DROP INDEX          | Remove an index                                      |
| EXPLAIN             | Show the query plan (estimated costs)                |
| EXPLAIN ANALYZE     | Run the query and show actual costs and timing       |
+---------------------+------------------------------------------------------+

Leftmost prefix rule (composite indexes):
  Index (a, b, c) helps queries on: a | a,b | a,b,c
  Does NOT help queries on: b | c | b,c

Index sweet spot:
  High cardinality + frequently filtered + < 10% of rows returned = ideal

Index anti-patterns:
  Boolean columns | Small tables | Bulk-write pipelines | Unused columns

Key insight:
  An index makes reads faster and writes slightly slower.
  The goal is the right index in the right place, not indexes everywhere.
```

---

**[Back to README](../README.md)**

**Prev:** [← Normalization](./normalization.md) &nbsp;|&nbsp; **Next:** [INNER JOIN →](../05_joins/inner_join.md)

**Related Topics:** [Tables and Constraints](./tables_and_constraints.md) · [Normalization](./normalization.md) · [Query Optimization](../08_performance/query_optimization.md)

---

## 📝 Practice Questions

- 📝 [Q38 · btree-index](../sql_practice_questions_100.md#q38--thinking--btree-index)
- 📝 [Q39 · composite-index-order](../sql_practice_questions_100.md#q39--critical--composite-index-order)
- 📝 [Q40 · partial-index](../sql_practice_questions_100.md#q40--design--partial-index)
- 📝 [Q41 · covering-index](../sql_practice_questions_100.md#q41--thinking--covering-index)
- 📝 [Q42 · index-when-not](../sql_practice_questions_100.md#q42--critical--index-when-not)
- 📝 [Q78 · explain-indexes](../sql_practice_questions_100.md#q78--interview--explain-indexes)
- 📝 [Q97 · design-index-strategy](../sql_practice_questions_100.md#q97--design--design-index-strategy)

