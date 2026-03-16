# SQL — Interview Questions

> Real questions from data engineering, backend, analytics, and DevOps interviews. Grouped by experience level. Each answer is thorough enough to satisfy an interviewer, with SQL examples where relevant.

---

## Beginner (0–1 Years)

---

**Q1. What is a primary key?**

A primary key is a column (or combination of columns) that uniquely identifies each row in a table. No two rows can share the same primary key value, and the column cannot be NULL. Primary keys also serve as the default join target when other tables reference this table via a foreign key. In PostgreSQL, defining a column as `PRIMARY KEY` automatically creates a unique index on it.

```sql
CREATE TABLE users (
    user_id   SERIAL PRIMARY KEY,   -- auto-incrementing, unique, not null
    email     TEXT   NOT NULL UNIQUE,
    name      TEXT   NOT NULL
);
```

---

**Q2. What is the difference between WHERE and HAVING?**

`WHERE` filters rows *before* any grouping or aggregation happens — it operates on individual rows. `HAVING` filters *after* `GROUP BY` aggregates the data — it operates on group summaries. You cannot use aggregate functions like `SUM()` or `COUNT()` in a `WHERE` clause; that's what `HAVING` is for.

```sql
-- WHERE: filters individual orders before grouping
-- HAVING: filters groups (customers) by their aggregate total
SELECT user_id, SUM(total_amount) AS revenue
FROM   orders
WHERE  status = 'completed'       -- individual row filter (before GROUP BY)
GROUP  BY user_id
HAVING SUM(total_amount) > 500;   -- group filter (after GROUP BY)
```

---

**Q3. What is NULL and how do you handle it?**

`NULL` represents the absence of a known value — it is not zero, it is not an empty string, it is "unknown." NULL is not equal to anything, including itself (`NULL = NULL` evaluates to NULL, not TRUE). To check for NULL, use `IS NULL` or `IS NOT NULL`. In calculations, NULL propagates: any arithmetic involving NULL returns NULL. Use `COALESCE(value, default)` to substitute a fallback when a value might be NULL.

```sql
SELECT name, COALESCE(phone, 'not provided') AS phone
FROM   users
WHERE  email IS NOT NULL;

-- This is WRONG (always returns no rows):
WHERE  phone = NULL

-- This is correct:
WHERE  phone IS NULL
```

---

**Q4. What does DISTINCT do?**

`DISTINCT` removes duplicate rows from the result set, returning only unique combinations of the selected columns. It is applied after the `SELECT` resolves column values. `DISTINCT` can be expensive on large datasets because the database must sort or hash all results to identify duplicates. Prefer `GROUP BY` if you need to combine deduplication with aggregation.

```sql
-- Without DISTINCT: returns a row for every order (many duplicates)
SELECT user_id FROM orders;

-- With DISTINCT: returns each user_id only once
SELECT DISTINCT user_id FROM orders;

-- DISTINCT on multiple columns: unique combinations of both
SELECT DISTINCT category, status FROM products;
```

---

**Q5. Explain INNER JOIN.**

An `INNER JOIN` returns only the rows where the join condition is satisfied in *both* tables. If a row exists in the left table but has no matching row in the right table (or vice versa), it is excluded from the results. This is the default join type and the most commonly used. It is equivalent to the set theory concept of intersection.

```sql
-- Returns only users who have placed at least one order
-- Users with no orders are excluded
SELECT u.name, o.order_id, o.total_amount
FROM   users u
INNER JOIN orders o ON o.user_id = u.user_id;
```

---

**Q6. What is the difference between CHAR, VARCHAR, and TEXT?**

`CHAR(n)` stores exactly n characters, padding with spaces if shorter. `VARCHAR(n)` stores up to n characters without padding — good when you want to enforce a length limit. `TEXT` stores strings of unlimited length. In PostgreSQL, `VARCHAR` and `TEXT` have identical performance; the length limit in `VARCHAR(n)` is the only meaningful difference. In MySQL, `TEXT` types are stored differently from `VARCHAR` and cannot have a default value or be fully indexed without a prefix length.

---

**Q7. What does ORDER BY do, and how do you sort by multiple columns?**

`ORDER BY` sorts the result set by one or more columns. The default direction is `ASC` (ascending). You can mix directions across columns. Sorting is applied near the end of query execution, after filtering and grouping.

```sql
SELECT name, department, salary
FROM   employees
ORDER  BY department ASC, salary DESC;
-- First sorted alphabetically by department,
-- then within each department, highest salary first
```

---

## Intermediate (1–3 Years)

---

**Q8. What is a CTE and when would you use it over a subquery?**

A CTE (Common Table Expression), written with the `WITH` keyword, is a named temporary result set that exists only for the duration of the query. It makes complex queries more readable by letting you break them into named steps. Use a CTE over a subquery when: (1) you need to reference the same subquery more than once, (2) the query has multiple logical steps that are easier to read as separate named blocks, or (3) you need recursion (CTEs support `WITH RECURSIVE`, subqueries do not). In PostgreSQL, CTEs are sometimes "optimisation fences" — the planner may not push predicates into them, so check EXPLAIN output for complex CTEs.

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(total_amount) AS revenue
    FROM   orders
    WHERE  status = 'completed'
    GROUP  BY 1
),
ranked_months AS (
    SELECT month, revenue,
           RANK() OVER (ORDER BY revenue DESC) AS rank
    FROM   monthly_revenue
)
SELECT * FROM ranked_months WHERE rank <= 3;
```

---

**Q9. Explain window functions with an example.**

Window functions perform calculations across a set of rows *related to the current row* without collapsing those rows into a single group (unlike aggregate functions). They use the `OVER()` clause to define the "window" of rows to consider. Common window functions: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`, `SUM() OVER()`, `AVG() OVER()`.

```sql
-- For each order, show the customer's running total and rank by spend
SELECT
    u.name,
    o.order_id,
    o.total_amount,
    SUM(o.total_amount) OVER (PARTITION BY o.user_id
                              ORDER BY o.created_at)   AS running_total,
    RANK()              OVER (ORDER BY o.total_amount DESC) AS spend_rank
FROM   users u
JOIN   orders o ON o.user_id = u.user_id
WHERE  o.status = 'completed';
```

`PARTITION BY` is like `GROUP BY` for the window — it resets the calculation per partition. `ORDER BY` inside `OVER()` defines the order within each partition.

---

**Q10. What is database normalization?**

Normalization is the process of structuring a relational database to reduce data redundancy and improve data integrity. It involves organising columns and tables to satisfy a series of "normal forms." The most commonly discussed are: **1NF** (each column holds atomic values, no repeating groups), **2NF** (no partial dependency — every non-key column depends on the whole primary key, not part of it), and **3NF** (no transitive dependency — non-key columns depend only on the primary key, not on other non-key columns). In practice, "normalized to 3NF" is the standard target for OLTP databases. Over-normalization can hurt read performance — data warehouses often intentionally denormalize (star/snowflake schema) for query speed.

---

**Q11. How does an index work?**

An index is a separate data structure — most commonly a B-tree — that stores a sorted copy of one or more columns with pointers to the corresponding table rows. When you query `WHERE email = 'alice@example.com'`, without an index the database reads every row (O(n) sequential scan). With an index on `email`, it performs a B-tree lookup: O(log n) comparisons to find the matching rows directly. The trade-off is that indexes consume disk space and must be updated on every INSERT, UPDATE, and DELETE, adding write overhead.

```sql
-- Without index: Seq Scan (reads all rows)
-- With index: Index Scan (jumps directly to matches)
CREATE INDEX idx_users_email ON users (email);
```

---

**Q12. What is the difference between DELETE and TRUNCATE?**

`DELETE` removes rows one by one and logs each deletion, which makes it slower but reversible (within a transaction) and trigger-aware. It supports a `WHERE` clause for selective deletion. `TRUNCATE` removes all rows at once by deallocating data pages — much faster for large tables. In PostgreSQL, `TRUNCATE` is transactional; in MySQL, it is not. `TRUNCATE` does not fire row-level triggers and resets `SERIAL`/`AUTO_INCREMENT` sequences when used with `RESTART IDENTITY`. Use `DELETE` for selective, auditable, trigger-compatible removal; use `TRUNCATE` to wipe a staging table quickly.

---

**Q13. What are the different types of JOINs?**

The main join types are: **INNER JOIN** — only rows with matches in both tables; **LEFT JOIN** — all rows from the left table, NULLs for non-matching right rows; **RIGHT JOIN** — all rows from the right table, NULLs for non-matching left rows; **FULL OUTER JOIN** — all rows from both tables, NULLs where no match; **CROSS JOIN** — every row from the left combined with every row from the right (Cartesian product); **SELF JOIN** — a table joined to itself (useful for hierarchical data). In practice, `INNER JOIN` and `LEFT JOIN` cover the vast majority of use cases.

---

**Q14. What is a subquery and where can it be used?**

A subquery is a SELECT statement nested inside another SQL statement. It can appear in: the `WHERE` clause (as a filter), the `FROM` clause (as a derived table), the `SELECT` clause (as a scalar subquery returning one value), or with `EXISTS`/`NOT EXISTS`. Subqueries in `WHERE` with `IN` can be slow on large datasets — consider rewriting as a `JOIN` or `EXISTS` for better performance.

```sql
-- Scalar subquery in SELECT: add average order value alongside each order
SELECT order_id, total_amount,
       (SELECT AVG(total_amount) FROM orders WHERE status = 'completed') AS avg_order
FROM   orders
WHERE  status = 'completed';
```

---

## Advanced (3+ Years)

---

**Q15. How do you find the second highest salary?**

Several approaches work. The most portable uses a subquery:

```sql
-- Method 1: subquery with MAX and NOT IN
SELECT MAX(salary) AS second_highest
FROM   employees
WHERE  salary < (SELECT MAX(salary) FROM employees);

-- Method 2: DENSE_RANK window function (handles ties correctly)
SELECT salary
FROM (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM   employees
) ranked
WHERE rnk = 2
LIMIT 1;
```

`DENSE_RANK()` is the more robust approach because it handles duplicate salaries correctly. If two employees share the top salary, `DENSE_RANK` still assigns rank 2 to the next distinct value.

---

**Q16. Explain ACID properties.**

**Atomicity**: a transaction is all-or-nothing. If any statement fails, all changes are rolled back — no partial work persists. **Consistency**: a transaction brings the database from one valid state to another, never violating constraints (NOT NULL, FK, CHECK). **Isolation**: concurrent transactions are isolated from each other's uncommitted changes. The degree of isolation is configurable (READ COMMITTED, REPEATABLE READ, SERIALIZABLE). **Durability**: once a transaction is committed, it survives crashes, power failures, and restarts — it has been written to durable storage (WAL in PostgreSQL). ACID is the foundation of reliable transactional systems.

---

**Q17. What are isolation levels and when would you use SERIALIZABLE?**

Isolation levels control how much one transaction can see of another's in-progress changes. **READ COMMITTED** (PostgreSQL default): each statement sees only committed data — the most practical level for OLTP. **REPEATABLE READ**: guarantees that if you read a row twice in a transaction you get the same value, even if another transaction commits a change between those reads. **SERIALIZABLE**: the strictest level — transactions run as if they were executed one after another, preventing phantom reads and serialization anomalies. Use SERIALIZABLE when correctness is non-negotiable and concurrent transactions could produce incorrect results if interleaved — for example, financial double-spend prevention or inventory reservation systems. The trade-off is higher contention and possible transaction retries on serialization failure.

---

**Q18. How would you optimise a slow query?**

A structured approach: (1) **Identify** the slow query via `pg_stat_statements` or slow query logs. (2) **Run** `EXPLAIN ANALYZE` to see the execution plan — look for sequential scans on large tables, bad row estimates, nested loops with large sets, and sorts spilling to disk. (3) **Check indexes** — is the relevant column indexed? Is the WHERE clause "sargable" (can the index be used)? Functions applied to indexed columns break index usage. (4) **Rewrite** the query if needed: replace `SELECT *` with specific columns, convert correlated subqueries to JOINs, push filters deeper into subqueries, use `EXISTS` instead of `IN` for large subquery results. (5) **Add or modify indexes** based on the plan — composite indexes, partial indexes, or covering indexes. (6) **Verify** the improvement with `EXPLAIN ANALYZE` again and test in production with `pg_stat_statements`.

---

**Q19. Explain the difference between a clustered and non-clustered index.**

A **clustered index** determines the physical order of rows stored on disk — the table data is stored sorted by the clustered index key. In MySQL/InnoDB, the primary key is always the clustered index. Looking up by the clustered index is extremely fast because related rows are physically adjacent. A **non-clustered index** is a separate structure that stores the indexed values with pointers (row IDs) back to the actual table rows. A table can have many non-clustered indexes. In PostgreSQL, all indexes are technically non-clustered (heap-based tables), but you can physically reorder a table by an index with `CLUSTER table_name USING index_name` — though this is a one-time sort, not maintained automatically.

---

**Q20. What is a covering index and why is it useful?**

A covering index includes all columns needed to satisfy a query — the filter columns AND the select columns. When the query planner can answer a query entirely from the index without touching the main table, it performs an **index-only scan**, which is significantly faster (no heap fetches). In PostgreSQL, use the `INCLUDE` clause to add non-filterable columns to a covering index without affecting sort order.

```sql
-- Query: category filter, fetch name + price
SELECT name, price FROM products WHERE category = 'Electronics';

-- Covering index: all three columns present
CREATE INDEX idx_products_cat_covering
ON products (category) INCLUDE (name, price);
-- Result: Index Only Scan — table never touched
```

---

**Q21. What is the difference between a view and a materialized view?**

A **view** is a stored query with a name — every time you SELECT from it, the underlying query executes and returns live data. It uses no extra storage and is always current. A **materialized view** stores the query result physically on disk. It is fast to query (like a real table, and can be indexed) but becomes stale — you must `REFRESH MATERIALIZED VIEW` to update it. Use regular views for: simplifying complex queries, enforcing security boundaries, ensuring freshness. Use materialized views for: expensive aggregations, reporting dashboards, ETL pipelines where slightly stale data is acceptable and query speed is critical. MySQL does not have native materialized views.

---

**Q22. How do transactions and locking work together?**

When a transaction modifies a row, PostgreSQL acquires a row-level lock on it. Other transactions trying to modify the same row must wait until the lock is released at `COMMIT` or `ROLLBACK`. PostgreSQL uses MVCC (Multi-Version Concurrency Control) — readers never block writers and writers never block readers, because each transaction sees a consistent snapshot of the data at a point in time. Deadlocks can occur when two transactions each hold a lock the other needs; PostgreSQL detects deadlocks automatically and rolls back one transaction. To explicitly lock rows for "read then update" patterns, use `SELECT ... FOR UPDATE`.

```sql
BEGIN;
SELECT balance FROM accounts WHERE account_id = 101 FOR UPDATE;
-- Row is now locked; other transactions must wait to modify it
UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;
COMMIT;
```

---

## Scenario Questions

---

**Q23. "The users table has 10 million rows and a query is slow. What do you do?"**

First, I'd run `EXPLAIN ANALYZE` on the query to see the execution plan — specifically looking for sequential scans, bad row estimates, and expensive sort operations. I'd check `pg_stat_statements` to confirm it's actually this query that's slow and not something upstream.

Next, I'd examine the `WHERE` clause. Are the filter columns indexed? Is any function applied to an indexed column (which would prevent index use)? Is the query using `SELECT *` when only a few columns are needed?

If the column is not indexed, I'd create an index — possibly a composite or partial index based on the query patterns. I'd run `EXPLAIN ANALYZE` again to confirm the index is used. I'd also check statistics freshness (`ANALYZE users`) to ensure the planner's estimates are accurate.

Finally, I'd look at the query logic itself: are there correlated subqueries that could be JOINs? Is there a LIMIT that could be applied earlier? Could a materialized view pre-compute an expensive aggregation?

```sql
-- Step 1: identify the plan
EXPLAIN ANALYZE SELECT name, email FROM users WHERE country = 'UK' AND is_active = TRUE;

-- Step 2: add an index if needed
CREATE INDEX idx_users_country_active ON users (country) WHERE is_active = TRUE;

-- Step 3: verify
EXPLAIN ANALYZE SELECT name, email FROM users WHERE country = 'UK' AND is_active = TRUE;
```

---

**Q24. "Design a schema for a Twitter-like application."**

The core entities are users, tweets, follows, likes, and hashtags. A minimal but production-aware schema:

```sql
CREATE TABLE users (
    user_id     SERIAL PRIMARY KEY,
    username    VARCHAR(50)  NOT NULL UNIQUE,
    display_name TEXT        NOT NULL,
    bio         TEXT,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ
);

CREATE TABLE tweets (
    tweet_id    BIGSERIAL PRIMARY KEY,
    user_id     INT          NOT NULL REFERENCES users(user_id),
    content     VARCHAR(280) NOT NULL,
    reply_to_id BIGINT       REFERENCES tweets(tweet_id),  -- self-referencing for threads
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    deleted_at  TIMESTAMPTZ
);

CREATE TABLE follows (
    follower_id INT NOT NULL REFERENCES users(user_id),
    followee_id INT NOT NULL REFERENCES users(user_id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    CHECK (follower_id <> followee_id)  -- can't follow yourself
);

CREATE TABLE likes (
    user_id     INT    NOT NULL REFERENCES users(user_id),
    tweet_id    BIGINT NOT NULL REFERENCES tweets(tweet_id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, tweet_id)
);

-- Indexes for common access patterns
CREATE INDEX idx_tweets_user_created   ON tweets (user_id, created_at DESC);
CREATE INDEX idx_tweets_reply          ON tweets (reply_to_id) WHERE reply_to_id IS NOT NULL;
CREATE INDEX idx_follows_followee      ON follows (followee_id);
CREATE INDEX idx_likes_tweet           ON likes (tweet_id);
```

Key decisions: `BIGSERIAL` for tweet IDs (tweets grow fast), composite primary keys on `follows` and `likes` (each pair is unique and serves as a natural dedup), soft delete with `deleted_at` to preserve data integrity, and indexes tuned to common queries (user timelines, reply threads, follower counts).

---

**Q25. "Find all customers who ordered in January but NOT in February — write the query."**

```sql
-- Method 1: EXCEPT (clean and readable)
SELECT DISTINCT user_id
FROM   orders
WHERE  created_at >= '2024-01-01' AND created_at < '2024-02-01'
  AND  status = 'completed'

EXCEPT

SELECT DISTINCT user_id
FROM   orders
WHERE  created_at >= '2024-02-01' AND created_at < '2024-03-01'
  AND  status = 'completed';

-- Method 2: LEFT JOIN + NULL check (more flexible if you need user details)
SELECT u.user_id, u.name, u.email
FROM   users u
JOIN   orders jan_orders
       ON jan_orders.user_id  = u.user_id
      AND jan_orders.created_at >= '2024-01-01'
      AND jan_orders.created_at <  '2024-02-01'
      AND jan_orders.status = 'completed'
LEFT   JOIN orders feb_orders
       ON feb_orders.user_id  = u.user_id
      AND feb_orders.created_at >= '2024-02-01'
      AND feb_orders.created_at <  '2024-03-01'
      AND feb_orders.status = 'completed'
WHERE  feb_orders.order_id IS NULL;  -- no February order found
```

The `EXCEPT` approach is concise and often the clearest. The `LEFT JOIN` approach is better when you need to retrieve columns from the `users` table alongside the result.

---

**Q26. "What would you do if two engineers accidentally ran the same INSERT and created duplicate rows?"**

First, identify the duplicates:

```sql
-- Find duplicate emails (or whatever the logical unique key is)
SELECT email, COUNT(*) AS cnt
FROM   users
GROUP  BY email
HAVING COUNT(*) > 1;
```

Then delete the extras, keeping the row with the lowest `user_id` (earliest insert):

```sql
DELETE FROM users
WHERE  user_id NOT IN (
    SELECT MIN(user_id)
    FROM   users
    GROUP  BY email
);
```

After cleanup, add a `UNIQUE` constraint so this can never happen again:

```sql
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE (email);
```

For future inserts that might race, use `INSERT ... ON CONFLICT DO NOTHING` or `ON CONFLICT DO UPDATE`.

---

## Quick Reference: Key Concepts

```
+-----------------------------+------------------------------------------------+
| Concept                     | One-line answer                                |
+-----------------------------+------------------------------------------------+
| Primary key                 | Uniquely identifies each row; NOT NULL         |
| Foreign key                 | Links a column to a PK in another table        |
| NULL                        | Unknown value; use IS NULL, not = NULL         |
| WHERE vs HAVING             | WHERE = before GROUP BY; HAVING = after        |
| INNER JOIN                  | Only rows with matches in both tables          |
| LEFT JOIN                   | All left rows; NULL for unmatched right rows   |
| DISTINCT                    | Remove duplicate rows from result              |
| GROUP BY                    | Collapse rows into groups for aggregation      |
| Window function             | Aggregate over rows without collapsing them    |
| CTE (WITH ...)              | Named temp result for readability / recursion  |
| Index                       | B-tree structure for fast lookup               |
| Covering index              | Index that holds all needed columns (no table  |
|                             | heap access required)                          |
| Normalization               | Structure data to reduce redundancy (1NF-3NF)  |
| ACID                        | Atomicity, Consistency, Isolation, Durability  |
| Isolation levels            | READ COMMITTED → REPEATABLE READ → SERIALIZABLE|
| View                        | Saved query; always fresh                      |
| Materialized view           | Stored query result; must be refreshed         |
| DELETE vs TRUNCATE          | DELETE = selective + logged; TRUNCATE = fast   |
| Soft delete                 | Set deleted_at instead of removing the row     |
| SQL injection               | Prevented by parameterised queries             |
+-----------------------------+------------------------------------------------+
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← SQL with Python](../09_real_world/sql_with_python.md) &nbsp;|&nbsp; **Next:** [SQL Scenarios →](./sql_scenarios.md)

**Related Topics:** [Window Functions](../03_aggregation/window_functions.md) · [Indexes](../04_schema_design/indexes.md) · [Transactions](../07_data_modification/transactions.md) · [Query Optimization](../08_performance/query_optimization.md) · [Views](../09_real_world/views.md) · [Joins](../05_joins/inner_join.md)
