# INNER JOIN — The Matchmaker

## The Analogy

Imagine a dating app that only shows you matches when **both people** have completed their profile. If you filled yours in but the other person never joined, you don't see them. If they joined but you didn't fill in your profile, they don't see you either. The only connections that appear are the ones where **both sides exist and can be linked**.

That's exactly what an INNER JOIN does. It looks at two tables, finds the rows that have a matching value in both, and returns **only those matched pairs**. Rows that don't have a partner on the other side are silently dropped.

---

## What INNER JOIN Does

```
   customers                orders
   ---------                ------
   id | name            id | customer_id | total
   ---+------           ---+-------------+------
    1 | Alice            1 |      1      | 49.99
    2 | Bob              2 |      1      | 120.00
    3 | Carol            3 |      2      | 15.50
    4 | Dave             4 |      9      | 88.00  ← customer_id 9 doesn't exist
                         5 |      3      | 200.00

  INNER JOIN on customers.id = orders.customer_id

           ┌──────────┬───────────────┐
  customers│          │  orders       │
           │  ░░░░░░░░│░░░░░░░░       │
           │  ░░MATCH░│░░MATCH░░      │
           │  ░░░░░░░░│░░░░░░░░       │
           └──────────┴───────────────┘

  Result: only rows where customer_id EXISTS in customers
  → Alice's 2 orders (1, 2), Bob's order (3), Carol's order (5)
  → Dave (no orders) is EXCLUDED
  → Order 4 (customer_id 9, no customer) is EXCLUDED
```

**Venn diagram:**

```
     customers          orders
   ┌───────────────────────────┐
   │           │░░░░░│         │
   │  Dave     │░1   │ ord #4  │
   │  (no ord) │░2   │(no cust)│
   │           │░3   │         │
   │           │░5   │         │
   │           │░░░░░│         │
   └───────────────────────────┘
                ^^^^^
           only this part
           is returned
```

---

## Basic INNER JOIN Syntax

```sql
SELECT
    c.id          AS customer_id,
    c.name        AS customer_name,
    o.id          AS order_id,
    o.total
FROM customers AS c
INNER JOIN orders AS o
    ON c.id = o.customer_id;
```

`INNER JOIN` and `JOIN` are identical — `INNER` is optional but makes the intent explicit.

**Result:**

```
 customer_id | customer_name | order_id | total
-------------+---------------+----------+--------
           1 | Alice         |        1 |  49.99
           1 | Alice         |        2 | 120.00
           2 | Bob           |        3 |  15.50
           3 | Carol         |        5 | 200.00
```

Dave is gone (no orders). Order #4 is gone (no matching customer).

---

## Setting Up Sample Tables

```sql
CREATE TABLE customers (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    city TEXT
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    product     TEXT,
    total       NUMERIC(10, 2),
    created_at  TIMESTAMPTZ DEFAULT now()
);

INSERT INTO customers (name, city) VALUES
    ('Alice',  'London'),
    ('Bob',    'Paris'),
    ('Carol',  'Berlin'),
    ('Dave',   'Madrid');   -- will have no orders

INSERT INTO orders (customer_id, product, total) VALUES
    (1, 'Laptop',     1299.00),
    (1, 'Mouse',        25.99),
    (2, 'Keyboard',    89.00),
    (3, 'Monitor',    349.00),
    (3, 'Webcam',      59.99);
-- intentionally no orders for Dave (id=4)
```

---

## The ON Clause

The `ON` clause defines **how the two tables relate**. It almost always links a foreign key in one table to a primary key in the other.

```sql
-- Explicit and readable (preferred)
FROM orders AS o
INNER JOIN customers AS c ON o.customer_id = c.id

-- You can also join on any expression
FROM orders AS o
INNER JOIN promotions AS p ON o.total >= p.min_spend
                           AND o.created_at BETWEEN p.starts_at AND p.ends_at
```

> **Tip:** Always alias your tables (`c`, `o`, `p`) and prefix every column with its alias. This eliminates ambiguity errors and makes queries self-documenting.

---

## What Gets Excluded

INNER JOIN is the most **restrictive** join type. A row is dropped from the result if:

1. The foreign key value is `NULL` (e.g., an order with no `customer_id`)
2. The referenced row doesn't exist (e.g., `customer_id = 9` but no customer with `id = 9`)
3. The joining column values simply don't match

```sql
-- This query will NOT return orders where customer_id IS NULL
SELECT c.name, o.total
FROM customers AS c
INNER JOIN orders AS o ON c.id = o.customer_id;
```

If you need to see rows even when there's no match, you want a LEFT JOIN (next file).

---

## Joining Multiple Tables

You can chain as many JOINs as needed. Each `JOIN … ON` adds another table to the result.

```sql
-- Three-table join: order line items with product names and customer names
SELECT
    c.name          AS customer,
    p.name          AS product,
    oi.quantity,
    oi.unit_price,
    (oi.quantity * oi.unit_price) AS line_total
FROM order_items AS oi
INNER JOIN orders     AS o  ON oi.order_id   = o.id
INNER JOIN customers  AS c  ON o.customer_id = c.id
INNER JOIN products   AS p  ON oi.product_id = p.id
WHERE o.created_at >= '2024-01-01'
ORDER BY c.name, o.id;
```

**Join chain visualised:**

```
order_items ──ON oi.order_id = o.id──► orders
                                          │
                              ON o.customer_id = c.id
                                          │
                                       customers

order_items ──ON oi.product_id = p.id──► products
```

Each `INNER JOIN` in the chain further filters: if any link is broken (no match), that row is dropped from the final result.

---

## Filtering After the Join

Add a `WHERE` clause to filter the already-joined rows:

```sql
-- Only London customers' orders over £100
SELECT c.name, o.product, o.total
FROM customers AS c
INNER JOIN orders AS o ON c.id = o.customer_id
WHERE c.city = 'London'
  AND o.total > 100.00
ORDER BY o.total DESC;
```

> **ON vs WHERE:** Put the join condition in `ON`; put row-filtering conditions in `WHERE`. Mixing them works but reduces readability and can trip you up with outer joins.

---

## Aggregating After a Join

Combining INNER JOIN with GROUP BY is extremely common — for example, total spend per customer:

```sql
SELECT
    c.id,
    c.name,
    COUNT(o.id)    AS order_count,
    SUM(o.total)   AS lifetime_value
FROM customers AS c
INNER JOIN orders AS o ON c.id = o.customer_id
GROUP BY c.id, c.name
ORDER BY lifetime_value DESC;
```

**Result:**

```
 id | name  | order_count | lifetime_value
----+-------+-------------+----------------
  3 | Carol |           2 |         408.99
  1 | Alice |           2 |        1324.99
  2 | Bob   |           1 |          89.00
```

Dave (no orders) doesn't appear — INNER JOIN already excluded him before GROUP BY ran.

---

## Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Forgetting the `ON` clause | Syntax error (PostgreSQL) or full cartesian product (old `FROM a, b` syntax) | Always include `ON` |
| Joining on `NULL` | Rows silently dropped — `NULL = NULL` is false in SQL | Use `IS NOT DISTINCT FROM` if NULLs should match |
| Ambiguous column names | Error: `column "id" is ambiguous` | Prefix every column with table alias |
| Expecting missing rows | INNER JOIN drops unmatched rows | Switch to LEFT JOIN if you need them |

---

## PostgreSQL vs MySQL Note

```
PostgreSQL:  INNER JOIN  (standard; INNER is optional)
MySQL:       same syntax — JOIN and INNER JOIN are identical
SQLite:      same syntax
SQL Server:  same syntax
```

All major databases support standard INNER JOIN syntax. No dialect differences to worry about here.

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                     INNER JOIN CHEAT SHEET                      │
├─────────────────────┬───────────────────────────────────────────┤
│ Keyword             │ INNER JOIN (INNER is optional)            │
│ Returns             │ Only rows with a match in BOTH tables     │
│ Unmatched rows      │ Silently excluded                         │
│ NULL foreign keys   │ Silently excluded                         │
│ Syntax              │ FROM a JOIN b ON a.id = b.a_id            │
│ Multiple tables     │ Chain additional JOIN … ON blocks         │
│ With aggregation    │ JOIN first, then GROUP BY                 │
│ Need unmatched rows?│ Use LEFT JOIN instead                     │
└─────────────────────┴───────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Indexes](../04_schema_design/indexes.md) &nbsp;|&nbsp; **Next:** [LEFT / RIGHT / OUTER JOIN →](./left_right_outer_join.md)

**Related Topics:** [LEFT/RIGHT/OUTER JOIN](./left_right_outer_join.md) · [CROSS and SELF JOIN](./cross_and_self_join.md) · [JOIN Patterns](./join_patterns.md)

---

## 📝 Practice Questions

- 📝 [Q23 · inner-join](../sql_practice_questions_100.md#q23--normal--inner-join)
- 📝 [Q79 · compare-joins](../sql_practice_questions_100.md#q79--interview--compare-joins)

