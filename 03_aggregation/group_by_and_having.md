# GROUP BY and HAVING

## The Sales Receipt Pile

Imagine you are the regional sales manager and someone drops 500 paper receipts on your desk. Your first instinct is to sort them into piles — one pile per city. London receipts here, Manchester there, Birmingham over there.

Once sorted, you count each pile and total up the revenue in it. Then your boss says: "I only care about cities that brought in more than £10,000 this month — throw the smaller piles away."

That three-step process — **sort into groups, aggregate each group, discard groups below a threshold** — is exactly what `GROUP BY` and `HAVING` do in SQL.

```
500 receipts (rows)
        |
        v
   [ GROUP BY city ]          <-- sort into piles
        |
    +--------+--------+--------+
    | London | Manch. | Birm.  |
    | £14k   | £8k    | £11k   |
    +--------+--------+--------+
        |
   [ HAVING revenue > 10000 ]  <-- discard small piles
        |
    +--------+--------+
    | London | Birm.  |    <-- final result
    +--------+--------+
```

---

## The Sample Schema

We will use a product orders table throughout this file.

```sql
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INT          NOT NULL,
    category    VARCHAR(50)  NOT NULL,
    product     VARCHAR(100) NOT NULL,
    amount      DECIMAL(10, 2),
    region      VARCHAR(50),
    created_at  DATE         NOT NULL
);
```

Sample data:

```
order_id | customer_id | category      | product            | amount  | region
---------|-------------|---------------|--------------------|---------|----------
1        | 101         | Electronics   | Wireless Keyboard  | 79.99   | North
2        | 102         | Electronics   | USB-C Hub          | 45.00   | South
3        | 101         | Furniture     | Monitor Stand      | 129.00  | North
4        | 103         | Electronics   | Webcam HD          | 89.99   | South
5        | 104         | Accessories   | Desk Lamp          | 34.00   | West
6        | 102         | Electronics   | Laptop Sleeve      | 34.50   | South
7        | 105         | Electronics   | Mechanical Kbd     | 149.99  | North
8        | 103         | Furniture     | Standing Desk      | 499.00  | West
9        | 106         | Furniture     | Ergonomic Chair    | 349.00  | North
10       | 107         | Accessories   | Cable Organiser    | 12.99   | South
11       | 108         | Electronics   | Webcam 4K          | 199.00  | North
12       | 109         | Accessories   | Phone Stand        | 18.50   | West
```

---

## GROUP BY — Collapsing Rows into Groups

`GROUP BY` collapses multiple rows that share the same value in a column into a single summary row. You must then use aggregate functions to represent the other columns.

### Single Column GROUP BY

Total revenue per product category:

```sql
SELECT
    category,
    COUNT(*)            AS total_orders,
    SUM(amount)         AS total_revenue,
    ROUND(AVG(amount), 2) AS avg_order_value
FROM orders
GROUP BY category;
```

```
category     | total_orders | total_revenue | avg_order_value
-------------|--------------|---------------|----------------
Accessories  | 3            | 65.49         | 21.83
Electronics  | 6            | 598.47        | 99.75
Furniture    | 3            | 977.00        | 325.67
```

The database groups all 12 rows into 3 buckets. Each bucket is represented by a single output row.

### The "non-aggregated column" Rule

Every column in `SELECT` that is **not** inside an aggregate function **must** appear in `GROUP BY`.

```sql
-- WRONG: product is not in GROUP BY and not aggregated
SELECT category, product, SUM(amount)
FROM orders
GROUP BY category;
-- ERROR: column "orders.product" must appear in the GROUP BY clause
--        or be used in an aggregate function

-- CORRECT: either add product to GROUP BY...
SELECT category, product, SUM(amount)
FROM orders
GROUP BY category, product;

-- ...or aggregate it
SELECT category, COUNT(DISTINCT product), SUM(amount)
FROM orders
GROUP BY category;
```

### Multiple Column GROUP BY

Break revenue down by both category and region:

```sql
SELECT
    category,
    region,
    COUNT(*)    AS orders,
    SUM(amount) AS revenue
FROM orders
GROUP BY category, region
ORDER BY category, region;
```

```
category     | region | orders | revenue
-------------|--------|--------|--------
Accessories  | South  | 1      | 12.99
Accessories  | West   | 2      | 52.50
Electronics  | North  | 3      | 428.98
Electronics  | South  | 3      | 169.49
Furniture    | North  | 2      | 478.00
Furniture    | West   | 1      | 499.00
```

Each unique combination of (category, region) becomes its own row.

---

## HAVING — Filtering Groups

`WHERE` filters individual rows **before** they are grouped. `HAVING` filters entire **groups** after aggregation.

### Show Only High-Revenue Categories

```sql
SELECT
    category,
    SUM(amount) AS total_revenue
FROM orders
GROUP BY category
HAVING SUM(amount) > 100;
```

```
category     | total_revenue
-------------|---------------
Electronics  | 598.47
Furniture    | 977.00
```

The `Accessories` group (£65.49) was formed and then discarded by `HAVING`.

### The Most Common Mistake — Using WHERE for Aggregates

This is something nearly every beginner gets wrong:

```sql
-- WRONG: WHERE cannot reference an aggregate function
SELECT category, SUM(amount) AS total_revenue
FROM orders
WHERE SUM(amount) > 100
GROUP BY category;

-- ERROR: aggregate functions are not allowed in WHERE
```

The reason this fails: `WHERE` runs **before** `GROUP BY`. At the time `WHERE` is evaluated, there are no groups yet — so there is nothing to aggregate.

Fix: replace `WHERE` with `HAVING`:

```sql
-- CORRECT
SELECT category, SUM(amount) AS total_revenue
FROM orders
GROUP BY category
HAVING SUM(amount) > 100;
```

---

## WHERE vs HAVING — Side by Side

Both filter rows but at completely different stages:

```
Query execution order:
  1. FROM      -- identify the table(s)
  2. WHERE     -- filter individual rows  <-- runs HERE (no groups yet)
  3. GROUP BY  -- form groups
  4. HAVING    -- filter groups           <-- runs HERE (groups exist)
  5. SELECT    -- compute output columns
  6. ORDER BY  -- sort the output
  7. LIMIT     -- restrict row count
```

A practical example using both together:

```sql
-- Revenue per category in the North region only,
-- for categories with more than 1 order
SELECT
    category,
    COUNT(*)    AS orders,
    SUM(amount) AS revenue
FROM orders
WHERE region = 'North'           -- filters rows BEFORE grouping
GROUP BY category
HAVING COUNT(*) > 1              -- filters groups AFTER grouping
ORDER BY revenue DESC;
```

```
category     | orders | revenue
-------------|--------|--------
Electronics  | 3      | 428.98
Furniture    | 2      | 478.00
```

`WHERE region = 'North'` removed all non-North rows first. Then `GROUP BY` grouped the remaining rows. Then `HAVING COUNT(*) > 1` removed groups with only one order.

---

## Real-World Examples

### Find Customers Who Spent More Than £200 Total

```sql
SELECT
    customer_id,
    COUNT(*)            AS orders_placed,
    SUM(amount)         AS lifetime_spend
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 200
ORDER BY lifetime_spend DESC;
```

```
customer_id | orders_placed | lifetime_spend
------------|---------------|---------------
103         | 2             | 588.99
106         | 1             | 349.00
109         | 1             | 349.00
```

### Find Products with More Than One Order

```sql
SELECT
    product,
    COUNT(*) AS times_ordered
FROM orders
GROUP BY product
HAVING COUNT(*) > 1;
```

```
product    | times_ordered
-----------|---------------
Webcam HD  | 1
(no results -- each product appears only once in sample data)
```

### Categories with Average Order Value Under £100

```sql
SELECT
    category,
    ROUND(AVG(amount), 2) AS avg_value
FROM orders
GROUP BY category
HAVING AVG(amount) < 100
ORDER BY avg_value;
```

```
category    | avg_value
------------|----------
Accessories | 21.83
Electronics | 99.75
```

---

## Grouping by Expressions

You can group by computed expressions, not just raw columns.

```sql
-- Group orders by month
SELECT
    DATE_TRUNC('month', created_at) AS month,
    COUNT(*)                         AS orders,
    SUM(amount)                      AS revenue
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;
```

> **MySQL note:** Use `DATE_FORMAT(created_at, '%Y-%m')` instead of `DATE_TRUNC`.
> **SQLite note:** Use `strftime('%Y-%m', created_at)`.

---

## Summary

```
+------------+----------------------------------------------------------+
| Clause     | Purpose                                 | Runs when?    |
+------------+-----------------------------------------+---------------+
| WHERE      | Filter individual rows                  | Before GROUP BY|
| GROUP BY   | Collapse rows into groups               | After WHERE   |
| HAVING     | Filter groups by aggregate condition    | After GROUP BY |
+------------+-----------------------------------------+---------------+

Golden rules:
  - Every SELECT column must either be in GROUP BY or inside an aggregate
  - Use WHERE for row-level filters (fast, uses indexes)
  - Use HAVING for aggregate filters (runs after grouping)
  - You CAN use both WHERE and HAVING in the same query
  - HAVING can reference aggregate functions directly (HAVING SUM(x) > 100)

Query execution order (memorise this):
  FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT
```

---

**[Back to README](../README.md)**

**Prev:** [← Aggregate Functions](./aggregate_functions.md) &nbsp;|&nbsp; **Next:** [Window Functions →](./window_functions.md)

**Related Topics:** [Aggregate Functions](./aggregate_functions.md) · [Window Functions](./window_functions.md)

---

## 📝 Practice Questions

- 📝 [Q16 · group-by](../sql_practice_questions_100.md#q16--critical--group-by)
- 📝 [Q17 · having-vs-where](../sql_practice_questions_100.md#q17--thinking--having-vs-where)
- 📝 [Q92 · predict-having-where](../sql_practice_questions_100.md#q92--logical--predict-having-where)
- 📝 [Q94 · debug-wrong-group-by](../sql_practice_questions_100.md#q94--debug--debug-wrong-group-by)

