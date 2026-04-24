# Query Optimization — Making Slow Queries Fast

## The Restaurant Kitchen Analogy

Imagine a restaurant. Customers (your application) send orders through the waiter (your query). The waiter is fine — fast, professional. But the kitchen (your database engine) is chaos: chefs are rummaging through every bin to find one ingredient, nothing is labelled, and the head chef is re-reading the whole recipe for every single dish.

The waiter isn't the problem. The kitchen is. Query optimization is about understanding what the kitchen is doing wrong and fixing it — before your customers start complaining about wait times.

---

## How the Database Engine Executes a Query

Before you can optimise, you need to understand the *order* in which SQL is actually processed. It's not the order you write it:

```
Written order:           Execution order:
  SELECT                   1. FROM
  FROM                     2. JOIN
  WHERE                    3. WHERE
  GROUP BY        --->     4. GROUP BY
  HAVING                   5. HAVING
  ORDER BY                 6. SELECT
  LIMIT                    7. DISTINCT
                           8. ORDER BY
                           9. LIMIT / OFFSET
```

This matters because:
- You can't use a SELECT alias in a WHERE clause (WHERE runs before SELECT)
- HAVING filters *after* GROUP BY aggregates are computed
- LIMIT is applied last — the engine still does all the earlier work first

---

## The N+1 Problem

This is the most common performance killer in applications that use SQL.

**The problem:** instead of fetching all the data you need in one query, your application fires one query to get a list, then one *additional* query per row to get details.

```
N+1 pattern (bad):

  Query 1: SELECT order_id FROM orders WHERE user_id = 1042;
  -- Returns 50 order IDs

  Query 2:  SELECT * FROM order_items WHERE order_id = 1;
  Query 3:  SELECT * FROM order_items WHERE order_id = 2;
  Query 4:  SELECT * FROM order_items WHERE order_id = 3;
  ...
  Query 51: SELECT * FROM order_items WHERE order_id = 50;

  Total: 51 queries. Each has network + parse + execute overhead.
```

**The fix:** one JOIN fetches everything in a single round-trip.

```sql
-- Bad (N+1 in application code):
-- for each order_id: SELECT * FROM order_items WHERE order_id = ?

-- Good (one query with a JOIN):
SELECT
    o.order_id,
    o.created_at,
    oi.product_id,
    oi.quantity,
    oi.unit_price
FROM   orders o
JOIN   order_items oi ON o.order_id = oi.order_id
WHERE  o.user_id = 1042;
```

---

## SELECT * vs Specific Columns

`SELECT *` is convenient for exploration, but harmful in production:

```
SELECT * problems:
  1. Transfers more bytes over the network
  2. Prevents "index-only scans" (index can't cover unknown columns)
  3. Breaks when someone adds/removes/reorders columns
  4. Forces the query planner to resolve all column metadata
```

```sql
-- Bad: fetches all 15 columns including large TEXT and JSONB fields
SELECT * FROM products WHERE category = 'Electronics';

-- Good: fetch only what you need
SELECT product_id, name, price, stock_qty
FROM   products
WHERE  category = 'Electronics';
```

---

## Avoid Functions on Indexed Columns in WHERE

This is a sneaky one. Wrapping an indexed column in a function forces a full table scan — the index becomes useless.

```sql
-- BAD: the index on created_at cannot be used
-- The database must evaluate DATE(created_at) for EVERY row
SELECT order_id, total_amount
FROM   orders
WHERE  DATE(created_at) = '2024-06-15';

-- GOOD: use a range comparison — the index on created_at IS used
SELECT order_id, total_amount
FROM   orders
WHERE  created_at >= '2024-06-15 00:00:00'
  AND  created_at <  '2024-06-16 00:00:00';
```

Same principle applies to other functions:

```sql
-- BAD: LOWER() prevents index use
WHERE  LOWER(email) = 'alice@example.com'

-- GOOD: store emails lowercase, or use a functional index
WHERE  email = 'alice@example.com'

-- Also bad:
WHERE  YEAR(created_at) = 2024    -- MySQL
WHERE  user_id + 0 = 1042         -- arithmetic on indexed column
```

---

## EXISTS vs IN vs JOIN Performance

All three can answer "give me rows from A where something is true in B" — but they perform differently at scale.

```sql
-- Scenario: find users who have placed at least one order

-- Using IN (materialises a potentially huge subquery result)
SELECT user_id, name FROM users
WHERE  user_id IN (SELECT user_id FROM orders);

-- Using EXISTS (short-circuits: stops as soon as one match is found)
SELECT user_id, name FROM users u
WHERE  EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.user_id
);

-- Using JOIN (fast with indexes, but may return duplicate users if
-- the user has multiple orders — use DISTINCT or a subquery)
SELECT DISTINCT u.user_id, u.name
FROM   users u
JOIN   orders o ON o.user_id = u.user_id;
```

```
Performance guide:
  EXISTS  -- best when the subquery is large; stops at first match
  IN      -- fine for small subquery results; bad with NULLs
  JOIN    -- fastest when indexes exist on both sides; watch for duplicates

Rule of thumb: use EXISTS for "does at least one match exist?"
               use JOIN when you need columns from both tables
```

---

## Slow Query vs Optimised Version (Side by Side)

Here's a real-world example: "Find the top 10 customers by revenue in the last 90 days."

```sql
-- SLOW VERSION
SELECT
    u.name,
    SUM(o.total_amount) AS revenue
FROM   users u, orders o          -- old-style implicit cross join!
WHERE  u.user_id = o.user_id
  AND  YEAR(o.created_at) >= 2024 -- function on indexed column
  AND  o.status != 'cancelled'
GROUP  BY u.name                   -- grouping by non-unique column
ORDER  BY revenue DESC
LIMIT  10;

-- FAST VERSION
SELECT
    u.user_id,
    u.name,
    SUM(o.total_amount) AS revenue
FROM   users u
JOIN   orders o ON o.user_id = u.user_id   -- explicit JOIN
WHERE  o.created_at >= NOW() - INTERVAL '90 days'  -- sargable range
  AND  o.status = 'completed'              -- equality is faster than !=
GROUP  BY u.user_id, u.name                -- group by PK + name
ORDER  BY revenue DESC
LIMIT  10;
```

Changes made:
- Explicit `JOIN` replaces implicit comma join
- Range filter replaces `YEAR()` function — index on `created_at` can be used
- Filter on equality (`= 'completed'`) rather than inequality where possible
- Group by the primary key (`user_id`) as well as name

---

## LIMIT Early When Exploring

If you're writing a subquery or CTE that you then filter further, push the LIMIT as deep as possible:

```sql
-- Inefficient: sorts ALL 2 million rows, then takes 10
SELECT * FROM events ORDER BY event_time DESC LIMIT 10;

-- With an index on event_time DESC, this is already fast.
-- But without one, add the index (see Indexing Strategies chapter).

-- In subqueries: filter before joining, not after
SELECT u.name, e.event_type
FROM   users u
JOIN   (
    SELECT user_id, event_type
    FROM   events
    WHERE  event_time > NOW() - INTERVAL '7 days'  -- filter first
    LIMIT  1000                                      -- limit early
) e ON e.user_id = u.user_id;
```

---

## Quick Optimisation Checklist

```
+---------------------------------------------+---------------------------------+
| Anti-Pattern                                | Fix                             |
+---------------------------------------------+---------------------------------+
| SELECT *                                    | List only needed columns        |
| Function on indexed column in WHERE         | Rewrite as sargable range       |
| N+1 queries in application loop            | Single JOIN or batch IN()       |
| No LIMIT on large exploratory queries       | Add LIMIT while developing      |
| IN (large subquery)                         | Use EXISTS or a JOIN            |
| ORDER BY on non-indexed column (big table)  | Add index or accept the cost    |
| Implicit cross join (FROM a, b WHERE a.id=b.id) | Use explicit JOIN syntax    |
+---------------------------------------------+---------------------------------+
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Transactions](../07_data_modification/transactions.md) &nbsp;|&nbsp; **Next:** [Execution Plans →](./execution_plans.md)

**Related Topics:** [Execution Plans](./execution_plans.md) · [Indexing Strategies](./indexing_strategies.md)

---

## 📝 Practice Questions

- 📝 [Q71 · slow-query-causes](../sql_practice_questions_100.md#q71--thinking--slow-query-causes)
- 📝 [Q73 · exists-in-join-perf](../sql_practice_questions_100.md#q73--thinking--exists-in-join-perf)
- 📝 [Q83 · slow-table-scenario](../sql_practice_questions_100.md#q83--design--slow-table-scenario)

