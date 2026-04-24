# Views — A Window Into Your Data

## The Saved Search Analogy

Imagine you work in a library and you frequently need "all books published after 2000 that are currently checked in and are in the Science section." You could type that search every time — or you could save it as a named search called `available_science_books`. Next time, you just open that saved search and see live, up-to-date results.

A **view** is exactly that: a saved query with a name. It behaves like a virtual table. When you query a view, the database runs the underlying query and returns the results. The data isn't stored twice — just the query definition is stored.

```
Without a view:
  SELECT * FROM orders o
  JOIN users u ON u.user_id = o.user_id
  WHERE u.deleted_at IS NULL
    AND o.status = 'completed';    <-- type this every time

With a view called completed_orders:
  SELECT * FROM completed_orders;  <-- done
```

---

## CREATE VIEW

```sql
-- Basic view: active users only (soft-delete filter)
CREATE VIEW active_users AS
SELECT user_id, name, email, created_at
FROM   users
WHERE  deleted_at IS NULL;
```

Now you can query `active_users` exactly like a table:

```sql
SELECT * FROM active_users WHERE name LIKE 'A%';

SELECT COUNT(*) FROM active_users;

SELECT au.name, o.total_amount
FROM   active_users au
JOIN   orders o ON o.user_id = au.user_id;
```

---

## CREATE OR REPLACE VIEW

Updates the view definition without dropping and recreating it. Avoids permission issues (dropping a view removes its grants).

```sql
CREATE OR REPLACE VIEW active_users AS
SELECT user_id, name, email, phone, created_at   -- added phone
FROM   users
WHERE  deleted_at IS NULL;
```

> Limitation: `CREATE OR REPLACE` can add columns to the end, but cannot remove columns or change column types. To do that, `DROP VIEW` and recreate.

---

## Real-World Example 1: Active Customers View

```sql
CREATE VIEW active_customers AS
SELECT
    u.user_id,
    u.name,
    u.email,
    u.created_at                              AS member_since,
    COUNT(o.order_id)                         AS total_orders,
    COALESCE(SUM(o.total_amount), 0)          AS lifetime_value,
    MAX(o.created_at)                         AS last_order_date
FROM   users u
LEFT   JOIN orders o
       ON o.user_id = u.user_id AND o.status = 'completed'
WHERE  u.deleted_at IS NULL
  AND  u.is_active = TRUE
GROUP  BY u.user_id, u.name, u.email, u.created_at;
```

Usage:

```sql
-- Find high-value customers quickly
SELECT name, email, lifetime_value
FROM   active_customers
WHERE  lifetime_value > 500
ORDER  BY lifetime_value DESC;

-- Churn risk: active customers who haven't ordered in 90 days
SELECT name, email, last_order_date
FROM   active_customers
WHERE  last_order_date < NOW() - INTERVAL '90 days'
   OR  last_order_date IS NULL;
```

---

## Real-World Example 2: Sales Dashboard View

```sql
CREATE VIEW daily_sales_summary AS
SELECT
    DATE(o.created_at)                        AS sale_date,
    COUNT(DISTINCT o.order_id)                AS orders_placed,
    COUNT(DISTINCT o.user_id)                 AS unique_customers,
    SUM(o.total_amount)                       AS gross_revenue,
    AVG(o.total_amount)                       AS avg_order_value,
    SUM(oi.quantity)                          AS items_sold
FROM   orders o
JOIN   order_items oi ON oi.order_id = o.order_id
WHERE  o.status = 'completed'
GROUP  BY DATE(o.created_at);
```

```sql
-- Dashboard query becomes trivial
SELECT *
FROM   daily_sales_summary
WHERE  sale_date >= CURRENT_DATE - INTERVAL '30 days'
ORDER  BY sale_date DESC;
```

---

## Updatable Views

Simple views (single table, no aggregation, no DISTINCT) are automatically updatable in PostgreSQL. You can INSERT, UPDATE, and DELETE through them.

```sql
-- Simple view — updatable
CREATE VIEW pending_orders AS
SELECT order_id, user_id, status, total_amount, created_at
FROM   orders
WHERE  status = 'pending';

-- This UPDATE goes through to the underlying orders table
UPDATE pending_orders
SET    status = 'processing'
WHERE  order_id = 50231;
```

Complex views (JOINs, aggregations, DISTINCT) are **not** automatically updatable. Use `INSTEAD OF` triggers or `WITH CHECK OPTION` for those use cases.

```sql
-- Prevent inserting rows through a view that would be invisible in the view
CREATE VIEW recent_orders AS
SELECT * FROM orders
WHERE  created_at > NOW() - INTERVAL '30 days'
WITH CHECK OPTION;  -- prevents inserting orders with old dates through this view
```

---

## Materialized Views (PostgreSQL)

A regular view runs its query every time you SELECT from it. A **materialized view** runs the query once and stores the result physically — like a cached snapshot.

```
Regular View:           Materialized View:
  Query runs at SELECT    Query runs at REFRESH
  Always fresh            Stale until refreshed
  No extra storage        Stores data on disk
  Fast to create          Can be indexed!
  Good for: simple        Good for: expensive aggregations,
  filters on small tables reporting queries, ETL pipelines
```

```sql
-- Create a materialized view for a heavy aggregation
CREATE MATERIALIZED VIEW product_sales_stats AS
SELECT
    p.product_id,
    p.name,
    p.category,
    COUNT(oi.order_item_id)    AS times_ordered,
    SUM(oi.quantity)           AS total_units_sold,
    SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM   products p
JOIN   order_items oi ON oi.product_id = p.product_id
JOIN   orders o ON o.order_id = oi.order_id
WHERE  o.status = 'completed'
GROUP  BY p.product_id, p.name, p.category;

-- Add an index on the materialized view (not possible on regular views)
CREATE INDEX idx_mvw_product_sales_revenue
ON product_sales_stats (total_revenue DESC);
```

### Refreshing a Materialized View

```sql
-- Manual refresh (blocks reads in older PostgreSQL)
REFRESH MATERIALIZED VIEW product_sales_stats;

-- Non-blocking refresh (requires a UNIQUE index on the view)
REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_stats;

-- Schedule this with pg_cron or your application's scheduler
```

> **MySQL note:** MySQL does not have materialized views natively. The equivalent is to create a regular table and populate it on a schedule with `INSERT ... SELECT` or a stored procedure.

---

## When to Use Views vs Materialized Views

```
+-------------------------+-----------------------------+---------------------------+
| Scenario                | Use                         | Why                       |
+-------------------------+-----------------------------+---------------------------+
| Hide complexity from    | Regular view                | Always fresh data         |
| application queries     |                             |                           |
+-------------------------+-----------------------------+---------------------------+
| Security: expose only   | Regular view                | Grant SELECT on view,     |
| certain columns to a    |                             | not the base table        |
| read-only role          |                             |                           |
+-------------------------+-----------------------------+---------------------------+
| Heavy aggregation query | Materialized view           | Pre-compute expensive     |
| for a dashboard         |                             | work; index the result    |
+-------------------------+-----------------------------+---------------------------+
| Nightly reporting ETL   | Materialized view           | Refresh once per day;     |
|                         |                             | fast all day              |
+-------------------------+-----------------------------+---------------------------+
| Real-time data required | Regular view or direct query| Materialized view would   |
|                         |                             | be stale                  |
+-------------------------+-----------------------------+---------------------------+
```

---

## Dropping a View

```sql
DROP VIEW active_customers;

-- IF EXISTS avoids an error if the view doesn't exist
DROP VIEW IF EXISTS active_customers;

-- CASCADE: also drop dependent views and rules
DROP VIEW IF EXISTS active_customers CASCADE;

-- Materialized view
DROP MATERIALIZED VIEW IF EXISTS product_sales_stats;
```

---

## Summary

```
+----------------------------+----------------------------------------------+
| Command                    | Effect                                       |
+----------------------------+----------------------------------------------+
| CREATE VIEW name AS ...    | Define a virtual table backed by a query    |
| CREATE OR REPLACE VIEW     | Update view definition in-place             |
| SELECT * FROM view_name    | Query the view (runs underlying query)      |
| DROP VIEW name             | Remove the view                             |
| CREATE MATERIALIZED VIEW   | Store query result physically               |
| REFRESH MATERIALIZED VIEW  | Re-run the query, update stored data        |
+----------------------------+----------------------------------------------+

Key distinctions:
  Regular view       = always fresh, no storage, can't be indexed
  Materialized view  = snapshot, uses storage, can be indexed, must be refreshed
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Indexing Strategies](../08_performance/indexing_strategies.md) &nbsp;|&nbsp; **Next:** [Stored Procedures →](./stored_procedures.md)

**Related Topics:** [Stored Procedures](./stored_procedures.md) · [Triggers](./triggers.md) · [SQL with Python](./sql_with_python.md)

---

## 📝 Practice Questions

- 📝 [Q65 · views](../sql_practice_questions_100.md#q65--design--views)
- 📝 [Q66 · materialized-views](../sql_practice_questions_100.md#q66--interview--materialized-views)
- 📝 [Q81 · compare-view-matview](../sql_practice_questions_100.md#q81--interview--compare-view-matview)

