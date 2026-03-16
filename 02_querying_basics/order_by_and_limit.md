# ORDER BY and LIMIT

## The Stack of Papers Analogy

Imagine a desk covered in 500 printed order receipts, thrown on in the order they arrived. Your
manager asks: "Show me the 10 most expensive orders from last month."

You'd do two things:
1. **Sort** the stack — put the highest-value receipts on top
2. **Take the top 10** — only hand over those, not all 500

That's exactly what `ORDER BY` and `LIMIT` do. Sort the result set, then cut it down to only
the rows you need.

```sql
-- The 10 most expensive orders from last month
SELECT id, customer_id, total_amount, created_at
FROM   orders
WHERE  created_at >= '2024-03-01'
  AND  created_at <  '2024-04-01'
ORDER  BY total_amount DESC
LIMIT  10;
```

---

## The Orders Table We'll Use

```
  TABLE: orders
  ┌─────┬─────────────┬──────────────┬────────────┬────────────┐
  │ id  │ customer_id │ total_amount │ status     │ created_at │
  ├─────┼─────────────┼──────────────┼────────────┼────────────┤
  │  1  │      1      │    149.99    │ delivered  │ 2024-01-05 │
  │  2  │      2      │    349.00    │ delivered  │ 2024-01-18 │
  │  3  │      1      │     49.50    │ refunded   │ 2024-02-02 │
  │  4  │      3      │    899.95    │ delivered  │ 2024-02-20 │
  │  5  │      4      │     22.00    │ pending    │ 2024-03-01 │
  │  6  │      2      │    220.00    │ shipped    │ 2024-03-08 │
  │  7  │      5      │    115.00    │ delivered  │ 2024-03-15 │
  │  8  │      1      │   1250.00    │ delivered  │ 2024-03-22 │
  │  9  │      3      │     78.50    │ pending    │ 2024-03-28 │
  │ 10  │      4      │    NULL      │ cancelled  │ 2024-03-30 │
  └─────┴─────────────┴──────────────┴────────────┴────────────┘
```

---

## ORDER BY — Sorting Results

`ORDER BY` goes after `WHERE` (if present) and specifies which column to sort by:

```sql
-- Cheapest orders first
SELECT id, total_amount, created_at
FROM   orders
ORDER  BY total_amount ASC;
```

```sql
-- Most recent orders first (default when you think "latest")
SELECT id, total_amount, created_at
FROM   orders
ORDER  BY created_at DESC;
```

- `ASC` = ascending (smallest → largest, oldest → newest, A → Z) — **this is the default**
- `DESC` = descending (largest → smallest, newest → oldest, Z → A)

If you omit `ASC` or `DESC`, the database uses `ASC`. Best practice: **always write it
explicitly** so your intent is unambiguous to other readers.

---

## Sorting by Multiple Columns

You can sort by multiple columns — the second column only breaks ties in the first:

```sql
-- Sort by status A-Z, then by total_amount highest to lowest within each status
SELECT id, status, total_amount
FROM   orders
ORDER  BY status ASC, total_amount DESC;
```

Result (conceptual):

```
  id  │ status     │ total_amount
  ────┼────────────┼─────────────
   8  │ cancelled  │   NULL
   1  │ delivered  │   1250.00
   4  │ delivered  │    899.95
   2  │ delivered  │    349.00
   7  │ delivered  │    115.00
   1  │ delivered  │    149.99
   5  │ pending    │     78.50
   9  │ pending    │     22.00
   6  │ shipped    │    220.00
   3  │ refunded   │     49.50
```

---

## NULLS FIRST and NULLS LAST

NULL values have no natural position in a sort order. PostgreSQL puts NULLs **last** in ASC
order and **first** in DESC order by default. You can override this:

```sql
-- NULLs at the bottom, even in DESC sort
SELECT id, total_amount
FROM   orders
ORDER  BY total_amount DESC NULLS LAST;

-- NULLs at the top, even in ASC sort
SELECT id, total_amount
FROM   orders
ORDER  BY total_amount ASC NULLS FIRST;
```

Result of `DESC NULLS LAST`:

```
  id  │ total_amount
  ────┼──────────────
   8  │    1250.00
   4  │     899.95
   2  │     349.00
   6  │     220.00
   7  │     115.00
   1  │     149.99
   9  │      78.50
   3  │      49.50
   5  │      22.00
  10  │      NULL      ← forced to bottom
```

> **MySQL note:** MySQL does not support `NULLS FIRST` / `NULLS LAST` syntax. Workaround:
> `ORDER BY total_amount IS NULL ASC, total_amount DESC` (treats NULL as 0 for sorting purposes).

---

## LIMIT — Cap the Number of Rows Returned

`LIMIT n` tells the database to return at most `n` rows. It's applied after all filtering and
sorting:

```sql
-- The 3 most expensive orders ever
SELECT id, customer_id, total_amount
FROM   orders
ORDER  BY total_amount DESC NULLS LAST
LIMIT  3;
```

Result: orders 8 ($1250), 4 ($899.95), 2 ($349).

> **SQL Server note:** SQL Server uses `TOP n` instead of `LIMIT`:
> `SELECT TOP 3 id, total_amount FROM orders ORDER BY total_amount DESC`

---

## OFFSET — Skip Rows for Pagination

`OFFSET n` skips the first `n` rows before returning. Combined with `LIMIT`, this powers
**pagination** — the "page 1, page 2, page 3..." pattern seen in every web application:

```sql
-- Page 1: orders 1–5
SELECT id, total_amount, created_at
FROM   orders
ORDER  BY created_at DESC
LIMIT  5 OFFSET 0;

-- Page 2: orders 6–10
SELECT id, total_amount, created_at
FROM   orders
ORDER  BY created_at DESC
LIMIT  5 OFFSET 5;

-- Page 3: orders 11–15
SELECT id, total_amount, created_at
FROM   orders
ORDER  BY created_at DESC
LIMIT  5 OFFSET 10;
```

The pattern: `OFFSET = (page_number - 1) * page_size`

```
  Page   LIMIT   OFFSET   Rows returned
  ─────────────────────────────────────
    1      5        0       rows 1–5
    2      5        5       rows 6–10
    3      5       10       rows 11–15
    n      5    (n-1)*5     ...
```

**Performance note:** `OFFSET` becomes slow on large tables because the database still has to
scan and discard all the skipped rows. For high-traffic pagination on millions of rows, use
**keyset pagination** (also called cursor pagination) instead — `WHERE id > last_seen_id LIMIT n`.

---

## Real-World: The 10 Most Recent Orders

Here's the exact query you'd write in a customer support dashboard to show recent activity:

```sql
SELECT
    o.id                                AS order_id,
    c.first_name || ' ' || c.last_name  AS customer_name,
    o.total_amount,
    o.status,
    o.created_at
FROM   orders    o
JOIN   customers c ON c.id = o.customer_id
WHERE  o.status != 'cancelled'
ORDER  BY o.created_at DESC
LIMIT  10;
```

Don't worry about `JOIN` yet — that's covered in detail in Section 5. The point here is seeing
`ORDER BY ... DESC` and `LIMIT` together in a realistic context.

---

## Sorting by Column Position (Avoid This)

SQL allows sorting by column number instead of name:

```sql
-- Sort by the 2nd column in the SELECT list
SELECT id, total_amount, created_at
FROM   orders
ORDER  BY 2 DESC;
```

This works but is **fragile** — if you reorder the `SELECT` columns, your sort breaks silently.
Always sort by column name in production code.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  ORDER BY AND LIMIT — KEY TAKEAWAYS                                  │
├──────────────────────────────────┬───────────────────────────────────┤
│  ORDER BY col ASC                │  Sort low → high (default)        │
│  ORDER BY col DESC               │  Sort high → low                  │
│  Multiple columns                │  Second col breaks ties in first  │
│  NULLS FIRST / NULLS LAST        │  PostgreSQL only — control NULLs  │
│  LIMIT n                         │  Return at most n rows            │
│  OFFSET n                        │  Skip first n rows                │
│  Pagination formula              │  OFFSET = (page - 1) * page_size  │
│  Large-table pagination          │  Use keyset (WHERE id > x) instead│
│  SQL Server equivalent           │  TOP n instead of LIMIT           │
└──────────────────────────────────┴───────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← WHERE and Filtering](./where_and_filtering.md) &nbsp;|&nbsp; **Next:** [DISTINCT and Aliases →](./distinct_and_aliases.md)

**Related Topics:** [SELECT and FROM](./select_and_from.md) · [WHERE and Filtering](./where_and_filtering.md) · [DISTINCT and Aliases](./distinct_and_aliases.md)
