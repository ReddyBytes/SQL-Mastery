# LEFT, RIGHT, and FULL OUTER JOIN — The Inclusive Matchmaker

## The Analogy

You're building an HR dashboard and your manager asks: "Show me every employee and the name of their manager." Simple enough — but what about the CEO? She has no manager. If you use an INNER JOIN, the CEO disappears from the report entirely, which is wrong.

What you really want is: **"Show ALL employees; for those who have a manager, fill in the manager's name. For those who don't, just leave it blank."**

That's a LEFT JOIN. It keeps every row from the left (main) table, and for rows that have no match on the right, it fills in `NULL`. The result is always at least as many rows as the left table alone.

---

## Three Flavours of Outer Join

```
LEFT JOIN                  RIGHT JOIN              FULL OUTER JOIN
─────────────────          ────────────────        ───────────────────────
All left rows              All right rows          All rows from both sides
+ matching right rows      + matching left rows    NULLs fill the gaps
Right = NULL if no match   Left = NULL if no match

  L   R                      L   R                   L   R
┌───┬─────┐               ┌─────┬───┐             ┌─────┬─────┐
│░░░│░░░░░│               │░░░░░│░░░│             │░░░░░│░░░░░│
│░░░│░░░░░│               │░░░░░│░░░│             │░░░░░│░░░░░│
│░░░│     │               │     │░░░│             │░░░░░│░░░░░│
│░░░│     │               │     │░░░│             │░░░░░│     │
└───┴─────┘               └─────┴───┘             │     │░░░░░│
 ^^^                               ^^^             └─────┴─────┘
Keep all                       Keep all             Keep all rows
from LEFT                      from RIGHT           from BOTH
```

---

## Sample Data

```sql
CREATE TABLE employees (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    manager_id INT REFERENCES employees(id),  -- self-referential
    department TEXT
);

INSERT INTO employees (id, name, manager_id, department) VALUES
    (1, 'Sarah Chen',    NULL, 'Executive'),   -- CEO, no manager
    (2, 'James Okafor',     1, 'Engineering'),
    (3, 'Priya Sharma',     1, 'Marketing'),
    (4, 'Luca Rossi',       2, 'Engineering'),
    (5, 'Anna Kowalski',    2, 'Engineering'),
    (6, 'Mohamed Ali',      3, 'Marketing');

CREATE TABLE customers (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    city TEXT
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    total       NUMERIC(10, 2),
    created_at  DATE
);

INSERT INTO customers (id, name, city) VALUES
    (1, 'Alice',  'London'),
    (2, 'Bob',    'Paris'),
    (3, 'Carol',  'Berlin'),
    (4, 'Dave',   'Madrid');   -- no orders yet

INSERT INTO orders (customer_id, total, created_at) VALUES
    (1, 49.99,  '2024-03-01'),
    (1, 120.00, '2024-03-15'),
    (2, 15.50,  '2024-03-20'),
    (3, 200.00, '2024-04-01');
```

---

## LEFT JOIN

**"Keep ALL rows from the left table; match what you can from the right."**

```sql
SELECT
    e.id,
    e.name         AS employee,
    m.name         AS manager_name
FROM employees AS e
LEFT JOIN employees AS m ON e.manager_id = m.id
ORDER BY e.id;
```

**Result:**

```
 id | employee      | manager_name
----+---------------+--------------
  1 | Sarah Chen    | NULL           ← CEO, no manager — row kept, NULL filled
  2 | James Okafor  | Sarah Chen
  3 | Priya Sharma  | Sarah Chen
  4 | Luca Rossi    | James Okafor
  5 | Anna Kowalski | James Okafor
  6 | Mohamed Ali   | Priya Sharma
```

Sarah Chen survives. With INNER JOIN she would vanish.

---

## LEFT JOIN to Find Missing Rows

The **anti-join** pattern is one of the most useful SQL idioms: find rows in the left table that have **no match at all** in the right table.

```sql
-- Find customers who have NEVER placed an order
SELECT
    c.id,
    c.name,
    c.city
FROM customers AS c
LEFT JOIN orders AS o ON c.id = o.customer_id
WHERE o.id IS NULL;           -- ← the key: NULL means "no match was found"
```

**How it works:**

```
customers LEFT JOIN orders

 c.id | c.name | o.id   | o.total
------+--------+--------+---------
    1 | Alice  |     1  |   49.99
    1 | Alice  |     2  |  120.00
    2 | Bob    |     3  |   15.50
    3 | Carol  |     4  |  200.00
    4 | Dave   |  NULL  |   NULL   ← WHERE o.id IS NULL picks this row

Result:
 id | name | city
----+------+-------
  4 | Dave | Madrid
```

> **Gotcha:** Use `WHERE right_table.primary_key IS NULL`, not `WHERE right_table.some_nullable_column IS NULL` — a nullable column could be NULL even when a match exists.

---

## RIGHT JOIN

RIGHT JOIN is the mirror image of LEFT JOIN: keep all rows from the **right** table.

```sql
-- All orders, with customer info (even if the customer row somehow went missing)
SELECT
    c.name    AS customer_name,
    o.id      AS order_id,
    o.total
FROM customers AS c
RIGHT JOIN orders AS o ON c.id = o.customer_id;
```

```
 customer_name | order_id | total
---------------+----------+--------
 Alice         |        1 |  49.99
 Alice         |        2 | 120.00
 Bob           |        3 |  15.50
 Carol         |        4 | 200.00
```

> **Practical note:** RIGHT JOIN is uncommon in the wild. Most developers flip the table order and use LEFT JOIN instead — it reads more naturally (left table = "the main thing I care about"). The two are logically equivalent.

```sql
-- These two queries return identical results:
FROM customers c RIGHT JOIN orders o ON c.id = o.customer_id
FROM orders o    LEFT  JOIN customers c ON o.customer_id = c.id
```

---

## FULL OUTER JOIN

FULL OUTER JOIN returns **every row from both tables**. Unmatched rows on either side get NULLs for the other side's columns.

```sql
-- All customers and all orders — matched where possible, NULLs where not
SELECT
    c.id     AS customer_id,
    c.name   AS customer_name,
    o.id     AS order_id,
    o.total
FROM customers AS c
FULL OUTER JOIN orders AS o ON c.id = o.customer_id
ORDER BY c.id NULLS LAST, o.id;
```

**Result (using our data with an orphan order added):**

```
 customer_id | customer_name | order_id | total
-------------+---------------+----------+--------
           1 | Alice         |        1 |  49.99
           1 | Alice         |        2 | 120.00
           2 | Bob           |        3 |  15.50
           3 | Carol         |        4 | 200.00
           4 | Dave          |     NULL |   NULL   ← customer with no orders
        NULL | NULL          |        5 | 888.00   ← orphan order (if it existed)
```

Use FULL OUTER JOIN when reconciling two data sources and you need to see **all discrepancies** in both directions.

> **MySQL note:** MySQL does NOT support `FULL OUTER JOIN` natively. Emulate it with `UNION`:
> ```sql
> SELECT * FROM a LEFT  JOIN b ON a.id = b.a_id
> UNION
> SELECT * FROM a RIGHT JOIN b ON a.id = b.a_id;
> ```

---

## Filtering on the Join vs WHERE

When using outer joins, the placement of filter conditions matters enormously.

```sql
-- WRONG: The WHERE clause turns the LEFT JOIN into an INNER JOIN
SELECT c.name, o.total
FROM customers AS c
LEFT JOIN orders AS o ON c.id = o.customer_id
WHERE o.total > 100;           -- eliminates rows where o.total IS NULL (i.e., Dave)

-- CORRECT: Move the filter into the ON clause to keep all left-side rows
SELECT c.name, o.total
FROM customers AS c
LEFT JOIN orders AS o ON  c.id = o.customer_id
                      AND o.total > 100;      -- Dave still appears, o.total = NULL
```

The difference:
```
WHERE o.total > 100   →  Dave's row dropped (NULL > 100 is false)
ON    o.total > 100   →  Dave's row kept, just no matching order rows
```

---

## Combining Outer Joins

```sql
-- All employees with their department budget (if dept has a budget row)
-- and their manager's name (if they have a manager)
SELECT
    e.name                  AS employee,
    m.name                  AS manager,
    d.annual_budget
FROM employees AS e
LEFT JOIN employees    AS m ON e.manager_id  = m.id
LEFT JOIN dept_budgets AS d ON e.department  = d.dept_name
ORDER BY e.id;
```

Chain outer joins just like inner joins — each preserves the rows of its left table.

---

## Quick Comparison

```
┌──────────────────┬────────────────────────────────────────────────┐
│ Join Type        │ Rows Returned                                  │
├──────────────────┼────────────────────────────────────────────────┤
│ INNER JOIN       │ Only rows matched in BOTH tables               │
│ LEFT JOIN        │ All left rows; NULLs for unmatched right cols  │
│ RIGHT JOIN       │ All right rows; NULLs for unmatched left cols  │
│ FULL OUTER JOIN  │ All rows from both; NULLs fill both sides      │
└──────────────────┴────────────────────────────────────────────────┘
```

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│              LEFT / RIGHT / FULL OUTER JOIN CHEAT SHEET              │
├──────────────────────────┬───────────────────────────────────────────┤
│ LEFT JOIN                │ Keep all left rows; right = NULL if absent│
│ RIGHT JOIN               │ Keep all right rows; left = NULL if absent│
│ FULL OUTER JOIN          │ Keep all rows from both sides             │
│ Anti-join pattern        │ LEFT JOIN … WHERE right.id IS NULL        │
│ Filter on outer join     │ Put selective filters in ON, not WHERE    │
│ RIGHT JOIN equivalent    │ Swap table order, use LEFT JOIN instead   │
│ MySQL FULL OUTER JOIN    │ Emulate with LEFT + RIGHT + UNION         │
└──────────────────────────┴───────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← INNER JOIN](./inner_join.md) &nbsp;|&nbsp; **Next:** [CROSS and SELF JOIN →](./cross_and_self_join.md)

**Related Topics:** [INNER JOIN](./inner_join.md) · [CROSS and SELF JOIN](./cross_and_self_join.md) · [JOIN Patterns](./join_patterns.md)
