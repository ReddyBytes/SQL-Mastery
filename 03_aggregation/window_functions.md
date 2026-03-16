# Window Functions

## The Running Scoreboard

Picture a live sports scoreboard during a tournament. It shows every player's individual score, their rank in the competition, how far behind the leader they are, and how their score changed from the last round — all without "collapsing" the list down to a single number.

That is exactly what window functions do in SQL. Regular aggregate functions (`SUM`, `AVG`, `COUNT`) collapse many rows into one. Window functions **keep every row** but add a new calculated column that looks across a set of related rows — the "window."

```
Regular GROUP BY (collapses rows):        Window function (keeps rows):

 customer | total_spend                    customer | order_amt | running_total
----------|------------                   ----------|-----------|---------------
 Alice    | 280.00                         Alice     | 79.99     | 79.99
 Bob      | 45.00                          Alice     | 129.00    | 208.99
 Carol    | 149.99                         Alice     | 71.01     | 280.00
                                           Bob       | 45.00     | 45.00
 3 rows out                                Carol     | 149.99    | 149.99
                                           5 rows out (still all there)
```

---

## The Sample Data

```sql
CREATE TABLE customer_orders (
    order_id    SERIAL PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(50),
    region      VARCHAR(20),
    order_date  DATE,
    amount      DECIMAL(10, 2)
);
```

```
order_id | customer_name | region  | order_date | amount
---------|---------------|---------|------------|--------
1        | Alice         | North   | 2024-01-05 | 79.99
2        | Bob           | South   | 2024-01-07 | 45.00
3        | Alice         | North   | 2024-01-10 | 129.00
4        | Carol         | North   | 2024-01-12 | 149.99
5        | David         | South   | 2024-01-14 | 34.00
6        | Bob           | South   | 2024-01-15 | 89.50
7        | Eve           | North   | 2024-01-18 | 220.00
8        | Carol         | North   | 2024-01-20 | 55.00
9        | Alice         | North   | 2024-02-01 | 71.01
10       | David         | South   | 2024-02-03 | 112.00
```

---

## The OVER() Clause — The Heart of Window Functions

Every window function uses `OVER()` to define the window (the set of rows it looks at). Without `OVER()`, you just have a regular aggregate.

```sql
-- Regular aggregate: one row result
SELECT SUM(amount) FROM customer_orders;

-- Window function: every row kept, total added as a column
SELECT
    customer_name,
    amount,
    SUM(amount) OVER() AS grand_total
FROM customer_orders;
```

```
customer_name | amount | grand_total
--------------|--------|------------
Alice         | 79.99  | 985.49
Bob           | 45.00  | 985.49
Alice         | 129.00 | 985.49
Carol         | 149.99 | 985.49
David         | 34.00  | 985.49
...
```

`OVER()` with nothing inside it means "the window is all rows." Every row sees the same grand total.

---

## PARTITION BY — Splitting into Groups Without Collapsing

`PARTITION BY` inside `OVER()` divides rows into groups — just like `GROUP BY` — but crucially, it does **not** reduce the number of output rows.

```
PARTITION BY region splits the window into two groups:

  North group:           South group:
  Alice    79.99         Bob      45.00
  Alice   129.00         David    34.00
  Carol   149.99         Bob      89.50
  Eve     220.00         David   112.00
  Carol    55.00

  Each group has its own running total, rank, etc.
  But ALL rows are returned — no collapsing happens.
```

```sql
-- Total spend per region, shown on every row
SELECT
    customer_name,
    region,
    amount,
    SUM(amount) OVER(PARTITION BY region) AS region_total
FROM customer_orders;
```

```
customer_name | region | amount | region_total
--------------|--------|--------|-------------
Alice         | North  | 79.99  | 633.99
Alice         | North  | 129.00 | 633.99
Carol         | North  | 149.99 | 633.99
Eve           | North  | 220.00 | 633.99
Carol         | North  | 55.00  | 633.99
Alice         | North  | 71.01  | 633.99
Bob           | South  | 45.00  | 280.50
David         | South  | 34.00  | 280.50
Bob           | South  | 89.50  | 280.50
David         | South  | 112.00 | 280.50
```

All 10 rows are returned. Each row now shows the total for its own region.

---

## ROW_NUMBER, RANK, DENSE_RANK

These assign a number to each row within a partition based on some ordering.

```sql
SELECT
    customer_name,
    region,
    amount,
    ROW_NUMBER()  OVER(PARTITION BY region ORDER BY amount DESC) AS row_num,
    RANK()        OVER(PARTITION BY region ORDER BY amount DESC) AS rank,
    DENSE_RANK()  OVER(PARTITION BY region ORDER BY amount DESC) AS dense_rank
FROM customer_orders;
```

```
customer_name | region | amount | row_num | rank | dense_rank
--------------|--------|--------|---------|------|------------
Eve           | North  | 220.00 | 1       | 1    | 1
Carol         | North  | 149.99 | 2       | 2    | 2
Alice         | North  | 129.00 | 3       | 3    | 3
Alice         | North  | 79.99  | 4       | 4    | 4
Alice         | North  | 71.01  | 5       | 5    | 5
Carol         | North  | 55.00  | 6       | 6    | 6
David         | South  | 112.00 | 1       | 1    | 1
Bob           | South  | 89.50  | 2       | 2    | 2
Bob           | South  | 45.00  | 3       | 3    | 3
David         | South  | 34.00  | 4       | 4    | 4
```

The difference between the three functions shows when there are **ties**:

```
Imagine two rows tied for 2nd place with the same score:

ROW_NUMBER:  1, 2, 3, 4   -- always unique; tie-breaking is arbitrary
RANK:        1, 2, 2, 4   -- tied rows share rank; next rank is skipped (gap)
DENSE_RANK:  1, 2, 2, 3   -- tied rows share rank; no gap in sequence
```

### Practical Use: Top-N Per Group

Get the single highest-spending order per region:

```sql
SELECT customer_name, region, amount
FROM (
    SELECT
        customer_name,
        region,
        amount,
        ROW_NUMBER() OVER(PARTITION BY region ORDER BY amount DESC) AS rn
    FROM customer_orders
) ranked
WHERE rn = 1;
```

```
customer_name | region | amount
--------------|--------|--------
Eve           | North  | 220.00
David         | South  | 112.00
```

This pattern — window function in a subquery, filter on rank in the outer query — is one of the most common real-world uses of window functions.

---

## Running Totals with ORDER BY in the Window

Adding `ORDER BY` inside `OVER()` changes `SUM` from a static total into a running total.

```sql
SELECT
    order_id,
    customer_name,
    order_date,
    amount,
    SUM(amount) OVER(ORDER BY order_date) AS running_total
FROM customer_orders
ORDER BY order_date;
```

```
order_id | customer_name | order_date | amount | running_total
---------|---------------|------------|--------|---------------
1        | Alice         | 2024-01-05 | 79.99  | 79.99
2        | Bob           | 2024-01-07 | 45.00  | 124.99
3        | Alice         | 2024-01-10 | 129.00 | 253.99
4        | Carol         | 2024-01-12 | 149.99 | 403.98
5        | David         | 2024-01-14 | 34.00  | 437.98
6        | Bob           | 2024-01-15 | 89.50  | 527.48
7        | Eve           | 2024-01-18 | 220.00 | 747.48
8        | Carol         | 2024-01-20 | 55.00  | 802.48
9        | Alice         | 2024-02-01 | 71.01  | 873.49
10       | David         | 2024-02-03 | 112.00 | 985.49
```

---

## LAG and LEAD — Looking at Neighbouring Rows

`LAG` retrieves a value from a **previous** row. `LEAD` retrieves a value from the **next** row. Both are invaluable for period-over-period comparisons.

```sql
-- Month-over-month revenue, showing previous month's amount and growth
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(amount) AS revenue
    FROM customer_orders
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    month,
    revenue,
    LAG(revenue)  OVER(ORDER BY month)                          AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER(ORDER BY month))
        / LAG(revenue) OVER(ORDER BY month) * 100,
    1)                                                          AS pct_change
FROM monthly
ORDER BY month;
```

```
month       | revenue | prev_month_revenue | pct_change
------------|---------|--------------------|------------
2024-01-01  | 757.48  | NULL               | NULL
2024-02-01  | 183.01  | 757.48             | -75.8
```

The January row has `NULL` for previous revenue because there is no earlier month. That is expected and correct.

`LAG` and `LEAD` accept an optional offset and default value:

```sql
LAG(revenue, 1, 0)   -- look back 1 row; return 0 if no row exists
LEAD(revenue, 2)     -- look forward 2 rows
```

---

## Combining PARTITION BY and ORDER BY

Partition and order together give you running totals **per group**:

```sql
SELECT
    customer_name,
    region,
    order_date,
    amount,
    SUM(amount) OVER(
        PARTITION BY region
        ORDER BY order_date
    ) AS regional_running_total
FROM customer_orders
ORDER BY region, order_date;
```

```
customer_name | region | order_date | amount | regional_running_total
--------------|--------|------------|--------|------------------------
Alice         | North  | 2024-01-05 | 79.99  | 79.99
Alice         | North  | 2024-01-10 | 129.00 | 208.99
Carol         | North  | 2024-01-12 | 149.99 | 358.98
Eve           | North  | 2024-01-18 | 220.00 | 578.98
Carol         | North  | 2024-01-20 | 55.00  | 633.98
Alice         | North  | 2024-02-01 | 71.01  | 704.99
Bob           | South  | 2024-01-07 | 45.00  | 45.00    <-- resets for South
David         | South  | 2024-01-14 | 34.00  | 79.00
Bob           | South  | 2024-01-15 | 89.50  | 168.50
David         | South  | 2024-02-03 | 112.00 | 280.50
```

The running total resets at the start of each region partition.

---

## Window Function Reference

```
+-------------+--------------------------------------------------------------+
| Function    | What it does                                                 |
+-------------+--------------------------------------------------------------+
| ROW_NUMBER()| Unique sequential number within partition (no ties)          |
| RANK()      | Rank with gaps for ties (1,2,2,4)                            |
| DENSE_RANK()| Rank without gaps for ties (1,2,2,3)                         |
| SUM(x)      | Running total when ORDER BY is present; grand total if not   |
| AVG(x)      | Running average or partition average                         |
| COUNT(x)    | Running count or partition count                             |
| LAG(x, n)   | Value from n rows before current row                         |
| LEAD(x, n)  | Value from n rows after current row                          |
| FIRST_VALUE | First value in the window frame                              |
| LAST_VALUE  | Last value in the window frame                               |
| NTILE(n)    | Divide rows into n equal buckets                             |
+-------------+--------------------------------------------------------------+

Syntax pattern:
  function_name() OVER (
      [PARTITION BY col1, col2]   -- optional grouping
      [ORDER BY col3 DESC]        -- optional ordering within partition
  )

Key difference vs GROUP BY:
  GROUP BY   → collapses rows, one row per group
  OVER()     → keeps all rows, adds computed column
```

> **MySQL note:** Window functions are supported from MySQL 8.0 onwards.
> **SQLite note:** Supported from SQLite 3.25 (2018) onwards.

---

**[Back to README](../README.md)**

**Prev:** [← GROUP BY and HAVING](./group_by_and_having.md) &nbsp;|&nbsp; **Next:** [Tables and Constraints →](../04_schema_design/tables_and_constraints.md)

**Related Topics:** [Aggregate Functions](./aggregate_functions.md) · [GROUP BY and HAVING](./group_by_and_having.md)
