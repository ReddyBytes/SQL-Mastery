# CTEs — Sticky Notes on a Whiteboard

## The Analogy

Imagine you're solving a complex problem at a whiteboard. You write an intermediate result on a sticky note, give it a name, and stick it up where everyone can see it. For the rest of the session, anyone can refer to "that sticky note" by name. When the session ends, you throw it away — it was only needed for that discussion.

A **Common Table Expression (CTE)** works exactly like that sticky note. You define a named result set at the top of your query using `WITH`, give it a meaningful name, and then reference it one or more times in the main query below. When the query finishes executing, the CTE disappears.

---

## Basic CTE Syntax

```sql
WITH cte_name AS (
    -- This is the CTE body: any valid SELECT
    SELECT customer_id, SUM(total) AS lifetime_spend
    FROM orders
    GROUP BY customer_id
)
-- Now reference it in the main query
SELECT
    c.name,
    cs.lifetime_spend
FROM customers AS c
INNER JOIN cte_name AS cs ON c.id = cs.customer_id
ORDER BY cs.lifetime_spend DESC;
```

The `WITH` keyword opens the CTE block. Everything between `AS (` and `)` is the named sub-query. After the closing `)`, you write the main query that consumes it.

---

## Setup

```sql
CREATE TABLE customers (
    id         SERIAL PRIMARY KEY,
    name       TEXT,
    city       TEXT,
    signed_up  DATE
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT,
    total       NUMERIC(10,2),
    created_at  DATE
);

INSERT INTO customers VALUES
    (1, 'Alice',  'London',  '2023-01-15'),
    (2, 'Bob',    'Paris',   '2023-03-22'),
    (3, 'Carol',  'Berlin',  '2023-06-01'),
    (4, 'Dave',   'Madrid',  '2024-01-10'),
    (5, 'Eve',    'London',  '2024-02-28');

INSERT INTO orders VALUES
    (1,  1,  49.99,  '2024-01-05'),
    (2,  1, 320.00,  '2024-02-10'),
    (3,  2,  15.50,  '2024-02-20'),
    (4,  3, 200.00,  '2024-03-01'),
    (5,  3, 450.00,  '2024-03-15'),
    (6,  5, 990.00,  '2024-04-01'),
    (7,  5, 120.00,  '2024-04-18');
```

---

## Chaining Multiple CTEs

Separate multiple CTEs with a comma. Each CTE can reference those defined before it.

```sql
WITH
-- Step 1: total spend per customer
customer_spend AS (
    SELECT
        customer_id,
        COUNT(id)      AS order_count,
        SUM(total)     AS lifetime_spend
    FROM orders
    GROUP BY customer_id
),

-- Step 2: classify customers into tiers using the spend CTE
customer_tiers AS (
    SELECT
        cs.customer_id,
        cs.lifetime_spend,
        CASE
            WHEN cs.lifetime_spend >= 500 THEN 'Gold'
            WHEN cs.lifetime_spend >= 100 THEN 'Silver'
            ELSE 'Bronze'
        END AS tier
    FROM customer_spend AS cs
),

-- Step 3: join back to customer names
final_report AS (
    SELECT
        c.name,
        c.city,
        ct.lifetime_spend,
        ct.tier
    FROM customers AS c
    INNER JOIN customer_tiers AS ct ON c.id = ct.customer_id
)

-- Main query — just SELECT from the last CTE
SELECT *
FROM final_report
ORDER BY lifetime_spend DESC;
```

**Result:**

```
 name  | city   | lifetime_spend | tier
-------+--------+----------------+--------
 Eve   | London |        1110.00 | Gold
 Carol | Berlin |         650.00 | Gold
 Alice | London |         369.99 | Silver
 Bob   | Paris  |          15.50 | Bronze
```

Each CTE is like one step on the whiteboard: `customer_spend` → `customer_tiers` → `final_report`. The logic is visible and testable at each stage.

---

## CTE vs Subquery — When to Use Each

```
CTE                                 Subquery
──────────────────────────────────  ──────────────────────────────────
Named and readable                  Anonymous and inline
Can be referenced multiple times    Can only be used once
Defined before the main query       Embedded inside the main query
Recursive CTEs possible             Recursive not possible
PostgreSQL can sometimes optimise   Sometimes faster (no materialisation)
Great for complex multi-step logic  Great for simple one-off filters
```

**Same query, two styles — compare readability:**

```sql
-- Subquery version (harder to scan)
SELECT c.name, sub.total_spend
FROM customers AS c
INNER JOIN (
    SELECT customer_id, SUM(total) AS total_spend
    FROM orders GROUP BY customer_id
) AS sub ON c.id = sub.customer_id
WHERE sub.total_spend > 300;

-- CTE version (clear intent)
WITH order_totals AS (
    SELECT customer_id, SUM(total) AS total_spend
    FROM orders GROUP BY customer_id
)
SELECT c.name, ot.total_spend
FROM customers AS c
INNER JOIN order_totals AS ot ON c.id = ot.customer_id
WHERE ot.total_spend > 300;
```

---

## Referencing a CTE Multiple Times

A subquery would have to be duplicated if used twice. A CTE is defined once and referenced as many times as needed.

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(total)                      AS revenue
    FROM orders
    GROUP BY 1
)
SELECT
    curr.month,
    curr.revenue                             AS this_month,
    prev.revenue                             AS last_month,
    curr.revenue - prev.revenue              AS change,
    ROUND((curr.revenue - prev.revenue)
          / NULLIF(prev.revenue, 0) * 100, 1) AS pct_change
FROM monthly_revenue AS curr
LEFT JOIN monthly_revenue AS prev            -- same CTE, second reference
    ON curr.month = prev.month + INTERVAL '1 month'
ORDER BY curr.month;
```

Using the CTE twice eliminates the need to write the aggregation logic twice.

---

## Recursive CTEs — Walking a Hierarchy

A recursive CTE has two parts joined by `UNION ALL`:

1. **Anchor member** — the starting row(s) (the CEO, the root node)
2. **Recursive member** — each subsequent level, referencing the CTE itself

```sql
-- Walk the employee hierarchy top-down
WITH RECURSIVE org_chart AS (

    -- Anchor: start with the CEO (no manager)
    SELECT
        id,
        name,
        manager_id,
        1 AS level,
        name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: find direct reports of the current level
    SELECT
        e.id,
        e.name,
        e.manager_id,
        oc.level + 1,
        oc.path || ' → ' || e.name
    FROM employees AS e
    INNER JOIN org_chart AS oc ON e.manager_id = oc.id
)

SELECT
    REPEAT('  ', level - 1) || name AS indented_name,
    level,
    path
FROM org_chart
ORDER BY path;
```

**Result:**

```
 indented_name       | level | path
---------------------+-------+------------------------------------------
 Sarah Chen          |     1 | Sarah Chen
   James Okafor      |     2 | Sarah Chen → James Okafor
     Anna Kowalski   |     3 | Sarah Chen → James Okafor → Anna Kowalski
     Luca Rossi      |     3 | Sarah Chen → James Okafor → Luca Rossi
   Priya Sharma      |     2 | Sarah Chen → Priya Sharma
     Mohamed Ali     |     3 | Sarah Chen → Priya Sharma → Mohamed Ali
```

> **PostgreSQL note:** `WITH RECURSIVE` is the required keyword. MySQL supports it since v8.0. SQLite supports it since v3.35.

---

## Real-World Example — Monthly Cohort Retention

Classic SaaS analysis: of customers who signed up in month M, how many were still placing orders in month M+1, M+2, etc.?

```sql
WITH
-- Step 1: each customer's signup cohort month
cohorts AS (
    SELECT
        id AS customer_id,
        DATE_TRUNC('month', signed_up) AS cohort_month
    FROM customers
),

-- Step 2: each customer's active months (months they placed an order)
activity AS (
    SELECT
        customer_id,
        DATE_TRUNC('month', created_at) AS active_month
    FROM orders
),

-- Step 3: for each customer, months since their cohort start
retention_base AS (
    SELECT
        co.cohort_month,
        EXTRACT(MONTH FROM AGE(ac.active_month, co.cohort_month))::INT AS months_since_signup,
        COUNT(DISTINCT ac.customer_id) AS active_customers
    FROM cohorts AS co
    INNER JOIN activity AS ac ON co.customer_id = ac.customer_id
    GROUP BY co.cohort_month, months_since_signup
)

SELECT
    cohort_month,
    months_since_signup,
    active_customers
FROM retention_base
ORDER BY cohort_month, months_since_signup;
```

Each CTE is one conceptual step. A reviewer can read `cohorts`, `activity`, `retention_base` in sequence and understand the logic without decoding nested subqueries.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CTE CHEAT SHEET                              │
├────────────────────────────┬─────────────────────────────────────────┤
│ Basic syntax               │ WITH name AS ( SELECT ... )             │
│ Multiple CTEs              │ Separate with commas; order matters     │
│ Recursive CTE              │ WITH RECURSIVE; anchor UNION ALL recur  │
│ Reference multiple times   │ CTE wins over subquery here             │
│ vs subquery (readability)  │ CTE wins for complex multi-step logic   │
│ vs subquery (performance)  │ Subquery sometimes faster (no materialise│
│ CTE materialisation        │ PostgreSQL may cache result (MATERIALIZED│
│                            │ / NOT MATERIALIZED hint in PG 12+)      │
│ Recursive guard            │ Add depth limit or cycle check to avoid  │
│                            │ infinite loops in bad data               │
└────────────────────────────┴─────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Subqueries](./subqueries.md) &nbsp;|&nbsp; **Next:** [CASE and Conditionals →](./case_and_conditionals.md)

**Related Topics:** [Subqueries](./subqueries.md) · [CASE and Conditionals](./case_and_conditionals.md) · [Window Functions](../03_aggregation/window_functions.md)

---

## 📝 Practice Questions

- 📝 [Q48 · cte-basics](../sql_practice_questions_100.md#q48--normal--cte-basics)
- 📝 [Q49 · recursive-cte](../sql_practice_questions_100.md#q49--design--recursive-cte)
- 📝 [Q50 · cte-vs-subquery](../sql_practice_questions_100.md#q50--interview--cte-vs-subquery)
- 📝 [Q77 · compare-cte-subquery](../sql_practice_questions_100.md#q77--interview--compare-cte-subquery)
- 📝 [Q84 · duplicate-rows-scenario](../sql_practice_questions_100.md#q84--design--duplicate-rows-scenario)
- 📝 [Q90 · cohort-retention](../sql_practice_questions_100.md#q90--design--cohort-retention)

