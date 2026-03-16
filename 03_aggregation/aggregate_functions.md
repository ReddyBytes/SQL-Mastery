# Aggregate Functions

## The Accountant's Monthly Report

Picture an accountant sitting at a desk with 10,000 paper invoices from the past month spread across the floor. Their job is not to hand all 10,000 sheets to the CEO — it is to summarise them: *How many invoices were there? What was the total revenue? What was the biggest single sale? What was the average order value?*

That is exactly what SQL aggregate functions do. They take a pile of rows and collapse them into a single meaningful number. Instead of retrieving every individual order, you ask the database: *give me the total, the count, the minimum, the maximum, the average* — all in one shot.

---

## The Sample Table

Throughout this file we will use a realistic `orders` table from an e-commerce platform.

```sql
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INT,
    product     VARCHAR(100),
    amount      DECIMAL(10, 2),
    status      VARCHAR(20),
    created_at  TIMESTAMP
);
```

Sample data:

```
order_id | customer_id | product          | amount  | status    | created_at
---------|-------------|------------------|---------|-----------|--------------------
1        | 101         | Wireless Keyboard| 79.99   | completed | 2024-01-05 09:12
2        | 102         | USB-C Hub        | 45.00   | completed | 2024-01-07 14:33
3        | 101         | Monitor Stand    | 129.00  | completed | 2024-01-10 11:05
4        | 103         | Webcam HD        | 89.99   | refunded  | 2024-01-12 16:20
5        | 104         | Desk Lamp        | NULL    | pending   | 2024-01-14 08:45
6        | 102         | Laptop Sleeve    | 34.50   | completed | 2024-01-15 13:10
7        | 105         | Mechanical Kbd   | 149.99  | completed | 2024-01-18 10:00
8        | 103         | Mouse Pad XL     | 22.00   | completed | 2024-01-20 17:30
```

Notice row 5: `amount` is NULL. This will matter a lot when we look at how each function handles missing data.

---

## COUNT — How Many Rows?

### COUNT(*)

`COUNT(*)` counts every row in the result set, regardless of what is in each column.

```sql
SELECT COUNT(*) AS total_orders
FROM orders;
```

```
total_orders
------------
8
```

The `*` means "count the row itself, not any particular column." NULL values in `amount` do not affect this count — the row exists, so it gets counted.

### COUNT(column)

`COUNT(column)` counts only rows where that specific column is **not NULL**.

```sql
SELECT COUNT(amount) AS orders_with_amount
FROM orders;
```

```
orders_with_amount
------------------
7
```

Row 5 has `amount = NULL`, so it is excluded. The result is 7, not 8.

This distinction is critical and is one of the most common interview questions:

```
COUNT(*)            = 8   -- counts every row
COUNT(amount)       = 7   -- skips the NULL row
COUNT(DISTINCT product) = 8   -- unique non-null products
```

> **MySQL / SQLite note:** `COUNT(*)` and `COUNT(col)` behave identically across these databases. The NULL-skipping rule for `COUNT(col)` is standard SQL and applies everywhere.

### Visualising the NULL Difference

```
orders table (8 rows)
+----------+-----------+------------------+---------+
| order_id | customer  | product          | amount  |
+----------+-----------+------------------+---------+
| 1        | 101       | Wireless Keyboard| 79.99   |  <-- counted by COUNT(*) AND COUNT(amount)
| 2        | 102       | USB-C Hub        | 45.00   |  <-- counted by COUNT(*) AND COUNT(amount)
| 3        | 101       | Monitor Stand    | 129.00  |  <-- counted by COUNT(*) AND COUNT(amount)
| 4        | 103       | Webcam HD        | 89.99   |  <-- counted by COUNT(*) AND COUNT(amount)
| 5        | 104       | Desk Lamp        | NULL    |  <-- COUNT(*) YES | COUNT(amount) NO
| 6        | 102       | Laptop Sleeve    | 34.50   |  <-- counted by COUNT(*) AND COUNT(amount)
| 7        | 105       | Mechanical Kbd   | 149.99  |  <-- counted by COUNT(*) AND COUNT(amount)
| 8        | 103       | Mouse Pad XL     | 22.00   |  <-- counted by COUNT(*) AND COUNT(amount)
+----------+-----------+------------------+---------+
COUNT(*)     = 8
COUNT(amount) = 7
```

---

## SUM — Add Them All Up

`SUM` adds up all non-NULL values in a numeric column.

```sql
SELECT SUM(amount) AS total_revenue
FROM orders;
```

```
total_revenue
-------------
550.47
```

The NULL in row 5 is silently ignored. SUM does not treat NULL as zero — it simply skips it.

Filter to only completed orders:

```sql
SELECT SUM(amount) AS completed_revenue
FROM orders
WHERE status = 'completed';
```

```
completed_revenue
-----------------
460.48
```

---

## AVG — The Mean Value

`AVG` divides the sum of non-NULL values by the count of non-NULL values.

```sql
SELECT AVG(amount) AS avg_order_value
FROM orders;
```

```
avg_order_value
---------------
78.638571...
```

Critically, AVG uses 7 (not 8) as its denominator because it excludes the NULL row. If you wanted to treat NULL as zero, you need to be explicit:

```sql
-- Treat NULL as 0 in the average
SELECT AVG(COALESCE(amount, 0)) AS avg_including_zero
FROM orders;
```

```
avg_including_zero
------------------
68.809...
```

The difference matters. Always ask: *should a missing amount count as zero, or should it be excluded entirely?*

---

## MIN and MAX — The Extremes

`MIN` and `MAX` return the smallest and largest non-NULL values respectively.

```sql
SELECT
    MIN(amount) AS cheapest_order,
    MAX(amount) AS most_expensive_order
FROM orders;
```

```
cheapest_order | most_expensive_order
---------------|---------------------
22.00          | 149.99
```

These also work on text and date columns, not just numbers:

```sql
-- Earliest and latest order
SELECT
    MIN(created_at) AS first_order,
    MAX(created_at) AS last_order
FROM orders;
```

```
first_order          | last_order
---------------------|--------------------
2024-01-05 09:12:00  | 2024-01-20 17:30:00
```

```sql
-- Alphabetically first and last product
SELECT
    MIN(product) AS first_alpha,
    MAX(product) AS last_alpha
FROM orders;
```

```
first_alpha       | last_alpha
------------------|------------------
Desk Lamp         | Wireless Keyboard
```

---

## Combining All Functions in One Query

This is where aggregate functions really shine — a single query that hands the accountant their full summary:

```sql
SELECT
    COUNT(*)                          AS total_orders,
    COUNT(amount)                     AS orders_with_amount,
    SUM(amount)                       AS total_revenue,
    ROUND(AVG(amount), 2)             AS avg_order_value,
    MIN(amount)                       AS min_order,
    MAX(amount)                       AS max_order
FROM orders
WHERE status = 'completed';
```

```
total_orders | orders_with_amount | total_revenue | avg_order_value | min_order | max_order
-------------|---------------------|---------------|-----------------|-----------|----------
6            | 6                   | 460.48        | 76.75           | 22.00     | 149.99
```

One row. Six numbers. The CEO is happy.

---

## NULL Behaviour — The Golden Rules

Every aggregate function in SQL follows these rules for NULLs:

```
+----------+-------------------------------------------------------+
| Function | NULL behaviour                                        |
+----------+-------------------------------------------------------+
| COUNT(*) | Counts every row — NULLs do not matter               |
| COUNT(c) | Skips rows where column c is NULL                    |
| SUM(c)   | Skips NULLs; returns NULL if ALL values are NULL     |
| AVG(c)   | Skips NULLs in both sum and count                    |
| MIN(c)   | Skips NULLs                                          |
| MAX(c)   | Skips NULLs                                          |
+----------+-------------------------------------------------------+
```

Edge case — what if every value is NULL?

```sql
-- If the entire column is NULL, SUM/AVG/MIN/MAX return NULL (not 0)
SELECT SUM(amount) FROM orders WHERE customer_id = 999;
-- Returns: NULL  (no rows matched)
```

Use `COALESCE` to convert a NULL result to 0 when needed:

```sql
SELECT COALESCE(SUM(amount), 0) AS safe_total
FROM orders
WHERE customer_id = 999;
-- Returns: 0
```

---

## COUNT DISTINCT — Unique Values Only

Count how many unique customers placed orders:

```sql
SELECT COUNT(DISTINCT customer_id) AS unique_customers
FROM orders;
```

```
unique_customers
----------------
5
```

Even though there are 8 orders, only 5 distinct customers placed them.

---

## Aggregate Functions with Filters

Aggregate functions work on whatever rows survive the `WHERE` clause. They never see the rows that were filtered out.

```sql
-- Revenue from January 10th onwards only
SELECT
    COUNT(*)       AS orders_after_10th,
    SUM(amount)    AS revenue_after_10th
FROM orders
WHERE created_at >= '2024-01-10';
```

```
orders_after_10th | revenue_after_10th
------------------|-------------------
5                 | 425.48
```

> **Important:** You cannot use aggregate functions directly inside a `WHERE` clause. To filter by aggregated values (e.g., "customers who spent more than $200"), you need `GROUP BY` + `HAVING`, covered in the next file.

---

## Summary

```
+------------------+---------------------------------------------+------------------+
| Function         | What it does                                | NULLs            |
+------------------+---------------------------------------------+------------------+
| COUNT(*)         | Count all rows                              | Included         |
| COUNT(col)       | Count non-NULL values in col                | Excluded         |
| COUNT(DISTINCT c)| Count unique non-NULL values                | Excluded         |
| SUM(col)         | Total of all non-NULL values                | Excluded         |
| AVG(col)         | Mean of non-NULL values                     | Excluded         |
| MIN(col)         | Smallest non-NULL value                     | Excluded         |
| MAX(col)         | Largest non-NULL value                      | Excluded         |
+------------------+---------------------------------------------+------------------+

Key rules:
  - COUNT(*) vs COUNT(col) differ only when NULLs are present
  - SUM/AVG/MIN/MAX all silently skip NULLs
  - Use COALESCE(SUM(col), 0) to safely handle all-NULL columns
  - WHERE filters rows BEFORE aggregation runs
  - Cannot use aggregate results in WHERE — use HAVING instead
```

---

**[Back to README](../README.md)**

**Prev:** [← DISTINCT and Aliases](../02_querying_basics/distinct_and_aliases.md) &nbsp;|&nbsp; **Next:** [GROUP BY and HAVING →](./group_by_and_having.md)

**Related Topics:** [GROUP BY and HAVING](./group_by_and_having.md) · [Window Functions](./window_functions.md)
