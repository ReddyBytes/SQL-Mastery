# Indexing Strategies — The Right Index for the Right Lookup

## The Textbook Analogy

A 600-page textbook without any indexes is a nightmare. To find everything about "database normalization," you'd have to read every page. But a good textbook has multiple indexes in the back:

- **Subject index**: look up "normalization" → pages 142, 198, 305
- **Author index**: look up "Codd, E.F." → pages 12, 47, 142
- **Glossary index**: look up terms alphabetically

Each index is optimised for a different type of lookup. Add too many and the book gets heavy and expensive to print. Add too few and readers give up. Database indexes work exactly the same way.

---

## What Is an Index?

An index is a separate data structure (typically a B-tree) that the database maintains alongside your table. It stores a sorted copy of one or more columns, plus a pointer back to the full row.

```
Table: products (10 million rows)

Without index on category:          With index on category:
  Scan all 10M rows                   Jump directly to 'Electronics'
  Filter for 'Electronics'            Return ~50,000 rows immediately
  Return ~50,000 rows

  Cost: O(n)                          Cost: O(log n)
```

The trade-off: indexes speed up reads but slow down writes (every INSERT/UPDATE/DELETE must also update the index). They also consume disk space.

---

## Creating a Basic Index

```sql
-- Single-column index: useful for filtering and sorting on this column
CREATE INDEX idx_products_category
ON products (category);

-- The database will now use this index for:
WHERE category = 'Electronics'
WHERE category IN ('Electronics', 'Office')
ORDER BY category
```

```sql
-- Index on a foreign key (almost always worth doing)
CREATE INDEX idx_orders_user_id
ON orders (user_id);

-- Without this, every JOIN between users and orders
-- causes a full scan of the orders table
```

---

## When to Add an Index

Use EXPLAIN ANALYZE to confirm whether an index helps:

```sql
-- Before: check the plan
EXPLAIN ANALYZE
SELECT product_id, name, price
FROM   products
WHERE  category = 'Electronics'
  AND  price < 50.00;

-- Output shows: Seq Scan on products (cost=0.00..24500.00 rows=1200000...)
-- That's a full scan on 1.2M rows — index needed!

-- Create the index
CREATE INDEX idx_products_category_price
ON products (category, price);

-- After: check again
EXPLAIN ANALYZE
SELECT product_id, name, price
FROM   products
WHERE  category = 'Electronics'
  AND  price < 50.00;

-- Now: Index Scan using idx_products_category_price
--      (cost=0.43..312.00 rows=3200...)
-- From 24500 cost units down to 312 — ~78x improvement!
```

---

## Composite Index Column Order — The Critical Rule

A composite (multi-column) index on `(A, B, C)` can be used for:
- Queries filtering on `A`
- Queries filtering on `A, B`
- Queries filtering on `A, B, C`

But NOT for:
- Queries filtering only on `B`
- Queries filtering only on `C`
- Queries filtering only on `B, C`

```
Index: (category, price)

               category    price
               --------    -----
               Electronics  12.99
               Electronics  29.99   <-- sorted by price WITHIN category
               Electronics  49.99
               Office        8.99
               Office       19.99
               Stationery    3.99

Usable for:   WHERE category = 'Electronics'
              WHERE category = 'Electronics' AND price < 50
              ORDER BY category, price

Not usable:   WHERE price < 50   (can't skip the first column)
```

**Rule: put the most selective (highest cardinality) column first, UNLESS you always filter on equality for one column — put that equality column first.**

```sql
-- Query pattern: always filter on status, sometimes filter on created_at
-- Put status first because it's used as an equality filter every time
CREATE INDEX idx_orders_status_created
ON orders (status, created_at);

-- This serves both:
WHERE status = 'completed'
WHERE status = 'completed' AND created_at > '2024-01-01'
```

---

## Covering Indexes (Index-Only Scans)

If your query only needs columns that are all present in the index, PostgreSQL can answer the query from the index alone — never touching the table. This is an **index-only scan** and is the fastest possible read.

```sql
-- Query: fetch name and price for Electronics products
SELECT name, price
FROM   products
WHERE  category = 'Electronics';

-- Covering index: includes all three columns the query touches
CREATE INDEX idx_products_covering
ON products (category, price, name);
-- Now: category = filter column, price + name = fetched columns
-- All needed data lives in the index — table not needed!
```

PostgreSQL's `INCLUDE` clause (v11+) lets you add non-filterable columns to a covering index without affecting the sort order:

```sql
CREATE INDEX idx_products_cat_price
ON products (category, price)
INCLUDE (name, stock_qty);
-- category and price are searchable/sortable
-- name and stock_qty are just "along for the ride" — covering bonus
```

---

## Partial Indexes

A partial index only indexes rows matching a condition. Smaller, faster, and often exactly what you need.

```sql
-- Only index active orders — completed/cancelled orders are never queried
CREATE INDEX idx_orders_pending
ON orders (user_id, created_at)
WHERE status = 'pending';

-- This index is tiny compared to indexing all orders
-- And it's used automatically when your query includes: WHERE status = 'pending'
```

```sql
-- Only index non-deleted users (soft-delete pattern)
CREATE INDEX idx_users_email_active
ON users (email)
WHERE deleted_at IS NULL;

-- Perfect for login lookups: WHERE email = ? AND deleted_at IS NULL
```

---

## Real-World: E-Commerce Product Search

For a product search page with filters and sorting, here's the indexing strategy:

```sql
-- Scenario: users filter by category, price range, sort by price or rating
-- Query:
SELECT product_id, name, price, avg_rating
FROM   products
WHERE  category    = 'Electronics'
  AND  price       BETWEEN 20 AND 100
  AND  is_active   = TRUE
ORDER  BY avg_rating DESC
LIMIT  20;

-- Strategy:
-- 1. Most queries filter on category (equality) and price (range)
CREATE INDEX idx_products_cat_price
ON products (category, price)
WHERE is_active = TRUE;           -- partial: only active products

-- 2. If sorting by avg_rating is common, include it
CREATE INDEX idx_products_cat_price_rating
ON products (category, price, avg_rating DESC)
WHERE is_active = TRUE;

-- 3. For product detail pages: single product_id lookup hits the PK index (free)
-- No extra index needed for WHERE product_id = ?
```

---

## When NOT to Index

More indexes is not always better. Avoid indexing:

```
+------------------------------------------+------------------------------------------+
| Situation                                | Why not to index                         |
+------------------------------------------+------------------------------------------+
| Low-cardinality columns                  | Boolean (TRUE/FALSE) or status with only |
| (e.g., is_active, gender)                | 2-3 values — index barely helps          |
+------------------------------------------+------------------------------------------+
| Heavily written tables                   | Every INSERT/UPDATE/DELETE updates all   |
| (e.g., event logs, metrics)              | indexes — write throughput suffers       |
+------------------------------------------+------------------------------------------+
| Small tables (< a few thousand rows)     | Seq scan is faster for tiny tables       |
+------------------------------------------+------------------------------------------+
| Columns never used in WHERE/JOIN/ORDER   | Dead weight — consumes disk and slows    |
|                                          | writes with no read benefit              |
+------------------------------------------+------------------------------------------+
```

---

## Index Bloat

As rows are inserted, updated, and deleted, indexes can accumulate dead pages — entries that point to deleted or superseded rows. This is **index bloat**.

```sql
-- Check index sizes and bloat (PostgreSQL)
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM   pg_stat_user_indexes
ORDER  BY pg_relation_size(indexrelid) DESC;

-- Rebuild a bloated index (locks the table briefly in PostgreSQL < 12)
REINDEX INDEX idx_orders_status_created;

-- Non-blocking rebuild (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_orders_status_created;
```

---

## Summary

```
+-------------------------+------------------------------------------------------+
| Index Type              | Best For                                             |
+-------------------------+------------------------------------------------------+
| Single-column           | Simple WHERE / ORDER BY on one column               |
| Composite (A, B)        | Queries filtering on A, or A+B together              |
| Covering                | Queries that only need indexed columns (no table I/O)|
| Partial (WHERE cond)    | Queries always filtered by a fixed condition         |
| Functional              | Queries using LOWER(col), EXTRACT(YEAR...), etc.     |
+-------------------------+------------------------------------------------------+

Column order rule:  equality columns first, range columns last
Bloat maintenance:  REINDEX CONCURRENTLY on high-write tables
When NOT to index:  low-cardinality, write-heavy, tiny tables
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Execution Plans](./execution_plans.md) &nbsp;|&nbsp; **Next:** [Views →](../09_real_world/views.md)

**Related Topics:** [Query Optimization](./query_optimization.md) · [Execution Plans](./execution_plans.md)
