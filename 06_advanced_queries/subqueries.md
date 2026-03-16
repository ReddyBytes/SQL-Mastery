# Subqueries — A Question Inside a Question

## The Analogy

Your manager asks: "Which customers spent more than the average customer last month?" To answer that, you first have to calculate the average — then compare each customer against it. You can't do both in one mental step; you do them in sequence.

A **subquery** lets you do exactly that in SQL. It's a query nested inside another query — the inner query runs first, produces a value (or a table), and the outer query uses that result. Think of it as asking a smaller question first so you can answer the bigger question.

---

## Four Places to Put a Subquery

```
1. In WHERE  → filter based on a calculated value
   SELECT ... FROM orders WHERE total > (SELECT AVG(total) FROM orders)

2. In FROM   → treat the result as a temporary table (derived table)
   SELECT * FROM (SELECT ...) AS subquery_name

3. In SELECT → compute a per-row value from another table
   SELECT name, (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) AS cnt

4. In HAVING → filter groups based on a calculated value
   SELECT dept, AVG(salary) FROM employees
   GROUP BY dept
   HAVING AVG(salary) > (SELECT AVG(salary) FROM employees)
```

---

## Setup

```sql
CREATE TABLE customers (
    id    SERIAL PRIMARY KEY,
    name  TEXT,
    city  TEXT
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT,
    total       NUMERIC(10,2),
    created_at  DATE
);

INSERT INTO customers VALUES
    (1, 'Alice',  'London'),
    (2, 'Bob',    'Paris'),
    (3, 'Carol',  'Berlin'),
    (4, 'Dave',   'Madrid'),
    (5, 'Eve',    'London');

INSERT INTO orders VALUES
    (1,  1,  49.99,  '2024-03-01'),
    (2,  1, 320.00,  '2024-03-15'),
    (3,  2,  15.50,  '2024-03-20'),
    (4,  3, 200.00,  '2024-04-01'),
    (5,  3, 450.00,  '2024-04-10'),
    (6,  4,  88.00,  '2024-04-12'),
    (7,  5, 990.00,  '2024-04-20');
```

---

## Subquery in WHERE — Scalar Subquery

A **scalar subquery** returns exactly one row and one column. It produces a single value that the outer query can compare against.

```sql
-- Find customers whose total spend exceeds the average
SELECT
    c.name,
    SUM(o.total) AS total_spend
FROM customers AS c
INNER JOIN orders AS o ON c.id = o.customer_id
GROUP BY c.id, c.name
HAVING SUM(o.total) > (
    SELECT AVG(total) FROM orders        -- scalar subquery: returns one number
)
ORDER BY total_spend DESC;
```

```
 name  | total_spend
-------+-------------
 Eve   |      990.00
 Carol |      650.00
 Alice |      369.99

-- AVG(total) = (49.99 + 320 + 15.50 + 200 + 450 + 88 + 990) / 7 ≈ 301.93
-- Bob (15.50) and Dave (88.00) are below average
```

---

## Subquery in FROM — Derived Table

A subquery in the `FROM` clause produces a **virtual table** that the outer query can treat like any real table. It must always have an alias.

```sql
-- Find customers whose single largest order exceeds £300
SELECT
    outer_q.customer_id,
    c.name,
    outer_q.max_order
FROM (
    SELECT
        customer_id,
        MAX(total) AS max_order
    FROM orders
    GROUP BY customer_id
) AS outer_q                             -- alias is mandatory
INNER JOIN customers AS c ON outer_q.customer_id = c.id
WHERE outer_q.max_order > 300
ORDER BY outer_q.max_order DESC;
```

```
 customer_id | name  | max_order
-------------+-------+-----------
           5 | Eve   |    990.00
           3 | Carol |    450.00
           1 | Alice |    320.00
```

> **Tip:** Derived tables are evaluated once and their result is passed to the outer query. For complex multi-step logic, a CTE (next file) is usually more readable.

---

## Subquery in SELECT — Correlated Subquery

A subquery in the `SELECT` list runs **once per row** of the outer query. It is called *correlated* because it references a column from the outer query (`c.id` below).

```sql
-- For each customer, show their name and total number of orders
SELECT
    c.id,
    c.name,
    (
        SELECT COUNT(*)
        FROM orders AS o
        WHERE o.customer_id = c.id        -- ← references outer query's c.id
    ) AS order_count
FROM customers AS c
ORDER BY order_count DESC;
```

```
 id | name  | order_count
----+-------+-------------
  1 | Alice |           2
  3 | Carol |           2
  2 | Bob   |           1
  4 | Dave  |           1
  5 | Eve   |           1
```

**The performance warning:** A correlated subquery in SELECT runs once for every row in the outer table. For 100,000 customers, that's 100,000 sub-executions. Rewrite as a JOIN when possible:

```sql
-- Equivalent but far more efficient on large tables
SELECT
    c.id,
    c.name,
    COUNT(o.id) AS order_count
FROM customers AS c
LEFT JOIN orders AS o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY order_count DESC;
```

---

## EXISTS vs IN

Both check whether a related row exists, but they behave differently.

**IN — checks if a value is in a list:**

```sql
-- Customers who have placed at least one order
SELECT c.name
FROM customers AS c
WHERE c.id IN (
    SELECT customer_id FROM orders
);
```

**EXISTS — checks if any matching row exists:**

```sql
-- Same result, different mechanism
SELECT c.name
FROM customers AS c
WHERE EXISTS (
    SELECT 1 FROM orders WHERE customer_id = c.id
);
```

**When to prefer EXISTS over IN:**

```
┌─────────────────────┬──────────────────────────────────────────────┐
│ Scenario            │ Recommendation                               │
├─────────────────────┼──────────────────────────────────────────────┤
│ Inner list is small │ IN is fine and often more readable           │
│ Inner list is large │ EXISTS is faster (stops at first match)      │
│ NULL values possible│ Use EXISTS — IN with NULLs gives surprises*  │
│ Correlated check    │ EXISTS (its natural form)                    │
└─────────────────────┴──────────────────────────────────────────────┘

* WHERE id NOT IN (SELECT customer_id FROM orders)
  returns NO rows if ANY customer_id in orders is NULL.
  Use NOT EXISTS to be safe.
```

---

## NOT EXISTS — Anti-Join Alternative

```sql
-- Customers who have NEVER placed an order (same as LEFT JOIN anti-join)
SELECT c.name
FROM customers AS c
WHERE NOT EXISTS (
    SELECT 1 FROM orders AS o WHERE o.customer_id = c.id
);
```

This is functionally equivalent to the LEFT JOIN pattern but can be faster in specific query plans — test both and check `EXPLAIN`.

---

## Subquery in HAVING

```sql
-- Departments whose average salary is above the company-wide average
SELECT
    dept,
    ROUND(AVG(salary), 2) AS avg_dept_salary
FROM employees
GROUP BY dept
HAVING AVG(salary) > (
    SELECT AVG(salary) FROM employees
)
ORDER BY avg_dept_salary DESC;
```

---

## Nested Subqueries

Subqueries can be nested multiple levels deep, though readability degrades quickly. Two levels is usually the practical maximum before a CTE is warranted.

```sql
-- Find the customer who placed the single most expensive order
SELECT name
FROM customers
WHERE id = (
    SELECT customer_id
    FROM orders
    WHERE total = (
        SELECT MAX(total) FROM orders   -- innermost runs first
    )
);
```

---

## When to Rewrite as a JOIN

| Situation | Keep Subquery | Rewrite as JOIN |
|---|---|---|
| Scalar value used once | Yes | Not needed |
| Correlated subquery in SELECT | No — slow | Yes |
| IN with small static list | Yes | Optional |
| Derived table used once | Sometimes | Use CTE for clarity |
| NOT IN with NULLs possible | Avoid | LEFT JOIN anti-join |

---

## Summary

```
┌───────────────────────────────────────────────────────────────────┐
│                      SUBQUERY CHEAT SHEET                         │
├───────────────────────┬───────────────────────────────────────────┤
│ Scalar (in WHERE)     │ Returns one value; compare with =, >, <   │
│ Derived table (FROM)  │ Virtual table; must have alias            │
│ Correlated (SELECT)   │ Runs per row — slow on large tables       │
│ EXISTS                │ Returns true/false; stops at first match  │
│ NOT EXISTS            │ Safer than NOT IN when NULLs possible     │
│ IN vs EXISTS          │ EXISTS better for large sets & NULLs      │
│ HAVING subquery       │ Compare group aggregates to a global value│
│ Performance tip       │ Correlated SELECT subquery → rewrite JOIN │
└───────────────────────┴───────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← JOIN Patterns](../05_joins/join_patterns.md) &nbsp;|&nbsp; **Next:** [CTEs →](./ctes.md)

**Related Topics:** [CTEs](./ctes.md) · [CASE and Conditionals](./case_and_conditionals.md) · [JOIN Patterns](../05_joins/join_patterns.md)
