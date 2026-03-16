# CASE and Conditionals — If-Else Inside a SELECT

## The Analogy

Imagine you're sorting a pile of customer orders into labelled trays: "Gold", "Silver", and "Bronze" depending on how much each customer has spent. You pick up each order, check the amount, and drop it in the right tray. You're applying a conditional rule to every item in the pile.

`CASE WHEN … THEN … ELSE … END` is exactly that — an if-else expression you can place inside a `SELECT` (or `ORDER BY`, or `GROUP BY`, or `HAVING`) to categorise or transform every row on the fly. No stored procedures, no application-side logic: the database does the sorting.

---

## Two Forms of CASE

### 1. Searched CASE — most flexible

Evaluates a separate boolean condition for each WHEN:

```sql
SELECT
    name,
    total_spend,
    CASE
        WHEN total_spend >= 1000 THEN 'Gold'
        WHEN total_spend >= 300  THEN 'Silver'
        WHEN total_spend >= 0    THEN 'Bronze'
        ELSE 'Unknown'
    END AS tier
FROM customer_summary;
```

Conditions are evaluated **top to bottom**; the first match wins. Once a branch matches, the rest are skipped — so ordering matters.

### 2. Simple CASE — like a switch statement

Compares a single expression to a list of values:

```sql
SELECT
    order_id,
    status,
    CASE status
        WHEN 'pending'   THEN 'Awaiting payment'
        WHEN 'paid'      THEN 'In fulfilment'
        WHEN 'shipped'   THEN 'On the way'
        WHEN 'delivered' THEN 'Complete'
        ELSE 'Unknown status'
    END AS status_label
FROM orders;
```

Simpler and slightly faster when you're matching one column to exact values.

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
    status      TEXT,
    created_at  DATE
);

INSERT INTO customers VALUES
    (1, 'Alice',  'London',  '2023-01-15'),
    (2, 'Bob',    'Paris',   '2023-03-22'),
    (3, 'Carol',  'Berlin',  '2023-06-01'),
    (4, 'Dave',   'Madrid',  '2024-01-10');

INSERT INTO orders VALUES
    (1, 1,  49.99, 'delivered', '2024-01-05'),
    (2, 1, 320.00, 'delivered', '2024-02-10'),
    (3, 2,  15.50, 'pending',   '2024-02-20'),
    (4, 3, 200.00, 'shipped',   '2024-03-01'),
    (5, 3, 450.00, 'delivered', '2024-03-15'),
    (6, 4,  88.00, 'cancelled', '2024-04-01');
```

---

## CASE in SELECT — Row-Level Classification

```sql
-- Bucket customers into spend tiers
SELECT
    c.name,
    c.city,
    SUM(o.total) AS lifetime_spend,
    CASE
        WHEN SUM(o.total) >= 500 THEN 'Gold'
        WHEN SUM(o.total) >= 100 THEN 'Silver'
        ELSE 'Bronze'
    END AS tier
FROM customers AS c
LEFT JOIN orders AS o ON c.id = o.customer_id
  AND o.status = 'delivered'
GROUP BY c.id, c.name, c.city
ORDER BY lifetime_spend DESC NULLS LAST;
```

```
 name  | city   | lifetime_spend | tier
-------+--------+----------------+--------
 Carol | Berlin |         650.00 | Gold
 Alice | London |         369.99 | Silver
 Dave  | Madrid |            NULL | Bronze
 Bob   | Paris  |            NULL | Bronze
```

---

## CASE in ORDER BY — Custom Sort Order

Standard `ORDER BY` sorts alphabetically or numerically. Use `CASE` to define your own priority.

```sql
-- Sort by urgency: pending first, then shipped, delivered last, cancelled at the bottom
SELECT
    id,
    customer_id,
    total,
    status
FROM orders
ORDER BY
    CASE status
        WHEN 'pending'   THEN 1
        WHEN 'shipped'   THEN 2
        WHEN 'delivered' THEN 3
        WHEN 'cancelled' THEN 4
        ELSE 5
    END,
    created_at;
```

---

## CASE in GROUP BY — Conditional Aggregation

```sql
-- Count orders by broad category: online vs in-store vs unknown
SELECT
    CASE
        WHEN channel IN ('web', 'mobile', 'app') THEN 'Online'
        WHEN channel IN ('shop', 'kiosk')        THEN 'In-store'
        ELSE 'Other'
    END AS channel_group,
    COUNT(*)    AS order_count,
    SUM(total)  AS revenue
FROM orders
GROUP BY channel_group
ORDER BY revenue DESC;
```

---

## Conditional Aggregation — CASE Inside SUM/COUNT

One of the most powerful patterns: aggregate only a subset of rows within each group.

```sql
-- Per customer: count delivered vs cancelled orders in one row
SELECT
    c.name,
    COUNT(o.id)                                          AS total_orders,
    COUNT(CASE WHEN o.status = 'delivered' THEN 1 END)   AS delivered,
    COUNT(CASE WHEN o.status = 'cancelled' THEN 1 END)   AS cancelled,
    SUM(CASE WHEN o.status = 'delivered' THEN o.total
             ELSE 0 END)                                 AS delivered_revenue
FROM customers AS c
LEFT JOIN orders AS o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY c.name;
```

```
 name  | total_orders | delivered | cancelled | delivered_revenue
-------+--------------+-----------+-----------+------------------
 Alice |            2 |         2 |         0 |            369.99
 Bob   |            1 |         0 |         0 |              0.00
 Carol |            2 |         1 |         0 |            450.00
 Dave  |            1 |         0 |         1 |              0.00
```

`COUNT(CASE WHEN … THEN 1 END)` counts only the rows where the condition is true — rows where the CASE returns NULL (the implicit ELSE) are not counted.

---

## COALESCE — First Non-NULL Value

`COALESCE` accepts any number of arguments and returns the first one that is not NULL. It's the go-to tool for supplying default values.

```sql
-- Use the nickname if available, fall back to first name, then 'Guest'
SELECT
    COALESCE(nickname, first_name, 'Guest') AS display_name
FROM users;

-- Replace NULL totals with 0 for arithmetic
SELECT
    c.name,
    COALESCE(SUM(o.total), 0) AS lifetime_spend
FROM customers AS c
LEFT JOIN orders AS o ON c.id = o.customer_id
GROUP BY c.name;
```

> **MySQL/SQLite note:** `COALESCE` is standard SQL and works everywhere. MySQL also has `IFNULL(a, b)` as a two-argument shorthand.

---

## NULLIF — Turn a Value Into NULL

`NULLIF(a, b)` returns NULL if `a = b`, otherwise returns `a`. Its main use is preventing division-by-zero errors.

```sql
-- Avoid division by zero when calculating conversion rate
SELECT
    campaign,
    clicks,
    conversions,
    ROUND(conversions::NUMERIC / NULLIF(clicks, 0) * 100, 2) AS conversion_pct
FROM ad_campaigns;

-- Without NULLIF, if clicks = 0, you get: ERROR: division by zero
-- With NULLIF(clicks, 0), clicks=0 becomes NULL, and NULL/anything = NULL
```

---

## CASE in HAVING

```sql
-- Departments where over half of employees are senior (salary > 70,000)
SELECT
    dept,
    COUNT(*) AS headcount
FROM employees
GROUP BY dept
HAVING
    SUM(CASE WHEN salary > 70000 THEN 1 ELSE 0 END)::NUMERIC
    / COUNT(*) > 0.5;
```

---

## Combining CASE with Window Functions

```sql
-- Flag if an order is above this customer's own average order value
SELECT
    o.id,
    o.customer_id,
    o.total,
    AVG(o.total) OVER (PARTITION BY o.customer_id) AS avg_for_customer,
    CASE
        WHEN o.total > AVG(o.total) OVER (PARTITION BY o.customer_id)
        THEN 'Above average'
        ELSE 'At or below average'
    END AS flag
FROM orders AS o;
```

---

## IIF — MySQL and SQL Server Shorthand

MySQL and SQL Server offer `IIF(condition, true_value, false_value)` as a shorthand for a two-branch CASE. PostgreSQL does not have `IIF` natively.

```sql
-- SQL Server / MySQL only
SELECT name, IIF(salary > 70000, 'Senior', 'Junior') AS grade FROM employees;

-- PostgreSQL equivalent
SELECT name, CASE WHEN salary > 70000 THEN 'Senior' ELSE 'Junior' END AS grade
FROM employees;
```

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│                  CASE AND CONDITIONALS CHEAT SHEET                   │
├────────────────────────────┬─────────────────────────────────────────┤
│ Searched CASE              │ CASE WHEN cond THEN val … ELSE … END    │
│ Simple CASE                │ CASE col WHEN val THEN result … END     │
│ CASE in SELECT             │ Classifies/transforms each row          │
│ CASE in ORDER BY           │ Custom sort priority                    │
│ CASE in GROUP BY           │ Group rows into dynamic buckets         │
│ Conditional aggregation    │ SUM/COUNT(CASE WHEN … THEN … END)       │
│ COALESCE(a, b, c)          │ Returns first non-NULL argument         │
│ NULLIF(a, b)               │ Returns NULL when a = b (÷ by zero fix) │
│ IIF(cond, t, f)            │ MySQL/SQL Server shorthand (2-branch)   │
│ Evaluation order           │ Top to bottom; first match wins         │
└────────────────────────────┴─────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← CTEs](./ctes.md) &nbsp;|&nbsp; **Next:** [String and Date Functions →](./string_and_date_functions.md)

**Related Topics:** [CTEs](./ctes.md) · [Subqueries](./subqueries.md) · [String and Date Functions](./string_and_date_functions.md)
