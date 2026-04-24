# JOIN Patterns — Common Recipes for Real-World Problems

## The Analogy

A good cookbook doesn't just list ingredients — it gives you **recipes**: specific combinations of techniques for specific outcomes. You learn once that "a roux is butter + flour stirred together before adding liquid", and then you use that pattern in a hundred different sauces.

JOIN patterns work the same way. The six patterns below are the roux of SQL. Once you recognise the shape of a problem, you reach for the matching recipe automatically — find unmatched rows, traverse a many-to-many, compare within a group, climb a hierarchy, etc.

---

## Sample Schema

All six patterns use these tables:

```sql
-- customers, orders, order_items, products, categories, employees
CREATE TABLE customers   (id SERIAL PRIMARY KEY, name TEXT, city TEXT, tier TEXT);
CREATE TABLE orders      (id SERIAL PRIMARY KEY, customer_id INT, created_at DATE, status TEXT);
CREATE TABLE order_items (id SERIAL PRIMARY KEY, order_id INT, product_id INT, qty INT, unit_price NUMERIC);
CREATE TABLE products    (id SERIAL PRIMARY KEY, name TEXT, category_id INT, price NUMERIC);
CREATE TABLE categories  (id SERIAL PRIMARY KEY, name TEXT);
CREATE TABLE employees   (id SERIAL PRIMARY KEY, name TEXT, manager_id INT, dept TEXT, salary NUMERIC);
```

---

## Pattern 1 — Find Unmatched Rows (Anti-Join)

**Problem:** Which customers have never placed an order?

```
  customers            orders
  ─────────            ──────
  Alice   ──────────►  order 1
  Bob     ──────────►  order 2
  Carol   ──────────►  order 3
  Dave    ──────────►  (nothing)
              ↑
        We want Dave
```

**Recipe: LEFT JOIN + WHERE right.id IS NULL**

```sql
SELECT
    c.id,
    c.name,
    c.city
FROM customers AS c
LEFT JOIN orders AS o ON c.id = o.customer_id
WHERE o.id IS NULL
ORDER BY c.name;
```

Why `o.id IS NULL`? When the LEFT JOIN finds no matching order, every `o.*` column comes back as NULL. Filtering on the primary key is the safest way to detect a true "no match" (unlike nullable columns, which could be NULL even when a match exists).

> **Alternative with NOT EXISTS** (sometimes faster on large tables):
> ```sql
> SELECT c.id, c.name
> FROM customers AS c
> WHERE NOT EXISTS (
>     SELECT 1 FROM orders WHERE customer_id = c.id
> );
> ```

---

## Pattern 2 — Many-to-Many via a Junction Table

**Problem:** Show every product in every order, with product names and order dates.

```
orders  ──────►  order_items  ──────►  products
   1                 (1, productA)         productA
   2                 (1, productB)         productB
                     (2, productA)
```

Many orders can contain many products. The junction table `order_items` holds the relationship.

**Recipe: Two INNER JOINs through the junction table**

```sql
SELECT
    o.id                            AS order_id,
    o.created_at,
    p.name                          AS product_name,
    oi.qty,
    oi.unit_price,
    (oi.qty * oi.unit_price)        AS line_total
FROM orders      AS o
INNER JOIN order_items AS oi ON o.id          = oi.order_id
INNER JOIN products    AS p  ON oi.product_id = p.id
WHERE o.status = 'completed'
ORDER BY o.id, p.name;
```

```
 order_id | created_at | product_name  | qty | unit_price | line_total
----------+------------+---------------+-----+------------+-----------
        1 | 2024-03-01 | Laptop        |   1 |    1299.00 |   1299.00
        1 | 2024-03-01 | Mouse         |   2 |      25.99 |     51.98
        2 | 2024-03-15 | Keyboard      |   1 |      89.00 |     89.00
```

---

## Pattern 3 — Non-Equality Join

**Problem:** Assign a tier label to each order based on its total value falling within a range table.

```
   order_totals     tier_rules
   ─────────────    ──────────────────────────────────
   order 1: £350    Bronze:  £0    – £99.99
   order 2: £1350   Silver:  £100  – £499.99
   order 3: £89     Gold:    £500  – £1999.99
                    Platinum: £2000+
```

**Recipe: JOIN with BETWEEN on the ON clause**

```sql
CREATE TABLE tier_rules (
    tier_name  TEXT,
    min_spend  NUMERIC,
    max_spend  NUMERIC
);
INSERT INTO tier_rules VALUES
    ('Bronze',   0,      99.99),
    ('Silver',   100,   499.99),
    ('Gold',     500,  1999.99),
    ('Platinum', 2000, 9999999);

SELECT
    o.id,
    o.created_at,
    SUM(oi.qty * oi.unit_price)  AS order_total,
    tr.tier_name
FROM orders      AS o
INNER JOIN order_items AS oi ON o.id = oi.order_id
INNER JOIN tier_rules  AS tr
    ON SUM(oi.qty * oi.unit_price) BETWEEN tr.min_spend AND tr.max_spend
GROUP BY o.id, o.created_at, tr.tier_name;
```

> **Note:** The above uses a range join. In PostgreSQL you can also use a lateral subquery or CASE expression for the same result. Range joins don't use standard B-tree indexes — keep this in mind for large tables.

---

## Pattern 4 — Multi-Table Join Chain

**Problem:** For each completed order, show the customer name, the product name, the category, and the line total.

**Recipe: Chain INNER JOINs — each table links to the next**

```sql
SELECT
    c.name                        AS customer,
    c.city,
    cat.name                      AS category,
    p.name                        AS product,
    oi.qty,
    ROUND(oi.qty * oi.unit_price, 2) AS line_total
FROM customers   AS c
INNER JOIN orders      AS o   ON c.id          = o.customer_id
INNER JOIN order_items AS oi  ON o.id          = oi.order_id
INNER JOIN products    AS p   ON oi.product_id = p.id
INNER JOIN categories  AS cat ON p.category_id = cat.id
WHERE o.status = 'completed'
  AND o.created_at >= '2024-01-01'
ORDER BY c.name, cat.name, p.name;
```

**Join chain diagram:**

```
customers → orders → order_items → products → categories
    c           o          oi           p          cat
    └── c.id = o.customer_id
                └── o.id = oi.order_id
                           └── oi.product_id = p.id
                                               └── p.category_id = cat.id
```

Each arrow is an INNER JOIN. Any broken link (missing foreign key match) removes that row from the result.

---

## Pattern 5 — Self-Referential Hierarchy

**Problem:** Show each employee alongside their manager's name and department.

**Recipe: LEFT JOIN the table to itself with two aliases**

```sql
SELECT
    e.id                           AS emp_id,
    e.name                         AS employee,
    e.dept,
    m.name                         AS manager,
    m.dept                         AS manager_dept,
    e.salary - m.salary            AS salary_diff
FROM employees AS e
LEFT JOIN employees AS m ON e.manager_id = m.id
ORDER BY m.name NULLS FIRST, e.name;
```

```
 emp_id | employee      | dept        | manager       | manager_dept | salary_diff
--------+---------------+-------------+---------------+--------------+------------
      1 | Sarah Chen    | Executive   | NULL          | NULL         |       NULL
      2 | James Okafor  | Engineering | Sarah Chen    | Executive    |     -15000
      3 | Priya Sharma  | Marketing   | Sarah Chen    | Executive    |     -20000
      4 | Luca Rossi    | Engineering | James Okafor  | Engineering  |     -12000
      5 | Anna Kowalski | Engineering | James Okafor  | Engineering  |     -18000
```

LEFT JOIN ensures the CEO (no manager) still appears. The `salary_diff` is NULL for the CEO because `m.salary` is NULL — which is correct behaviour.

---

## Pattern 6 — Joining with Aggregation

**Problem:** For each customer, show their total lifetime spend and most recent order date — include customers with no orders (show 0 and NULL).

**Recipe: LEFT JOIN + GROUP BY + aggregate functions**

```sql
SELECT
    c.id,
    c.name,
    c.city,
    COUNT(o.id)                   AS order_count,
    COALESCE(SUM(oi.qty * oi.unit_price), 0) AS lifetime_spend,
    MAX(o.created_at)             AS last_order_date
FROM customers   AS c
LEFT JOIN orders      AS o  ON c.id          = o.customer_id
LEFT JOIN order_items AS oi ON o.id          = oi.order_id
GROUP BY c.id, c.name, c.city
ORDER BY lifetime_spend DESC;
```

```
 id | name  | city   | order_count | lifetime_spend | last_order_date
----+-------+--------+-------------+----------------+-----------------
  1 | Alice | London |           2 |        1374.98 | 2024-03-15
  3 | Carol | Berlin |           1 |         349.00 | 2024-04-01
  2 | Bob   | Paris  |           1 |          89.00 | 2024-03-20
  4 | Dave  | Madrid |           0 |           0.00 | NULL
```

Key points:
- `LEFT JOIN` keeps Dave even with no orders.
- `COALESCE(SUM(...), 0)` converts NULL (no items) to 0 for the spend column.
- `COUNT(o.id)` counts non-NULL order IDs — Dave correctly gets 0.

---

## Pattern Quick-Reference

```
┌──────────────────────────────┬─────────────────────────────────────────────────┐
│ Pattern                      │ Recipe                                          │
├──────────────────────────────┼─────────────────────────────────────────────────┤
│ 1. Unmatched rows (anti-join)│ LEFT JOIN … WHERE right.id IS NULL              │
│ 2. Many-to-many              │ Two INNER JOINs through junction table          │
│ 3. Non-equality join         │ JOIN with BETWEEN / < / > in ON clause          │
│ 4. Multi-table chain         │ Chain INNER JOINs; each links to next table     │
│ 5. Self-referential hierarchy│ LEFT JOIN table AS e ON e.manager_id = m.id    │
│ 6. Join + aggregation        │ LEFT JOIN + GROUP BY + COALESCE(SUM(...), 0)    │
└──────────────────────────────┴─────────────────────────────────────────────────┘
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────────┐
│                      JOIN PATTERNS CHEAT SHEET                     │
├──────────────────────────┬─────────────────────────────────────────┤
│ Anti-join                │ LEFT JOIN + WHERE right PK IS NULL      │
│ Many-to-many             │ Double JOIN through junction table       │
│ Range / non-equality     │ JOIN ON value BETWEEN min AND max        │
│ Chain joins              │ Each table links via FK to the next      │
│ Self-referential         │ Same table with 2 aliases (e + m)        │
│ Aggregation with missing │ LEFT JOIN + COALESCE on SUM/COUNT        │
│ NOT EXISTS alternative   │ Often faster than anti-join on big sets  │
└──────────────────────────┴─────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← CROSS and SELF JOIN](./cross_and_self_join.md) &nbsp;|&nbsp; **Next:** [Subqueries →](../06_advanced_queries/subqueries.md)

**Related Topics:** [INNER JOIN](./inner_join.md) · [LEFT/RIGHT/OUTER JOIN](./left_right_outer_join.md) · [CROSS and SELF JOIN](./cross_and_self_join.md)

---

## 📝 Practice Questions

- 📝 [Q28 · anti-join](../sql_practice_questions_100.md#q28--thinking--anti-join)
- 📝 [Q29 · junction-table](../sql_practice_questions_100.md#q29--design--junction-table)

