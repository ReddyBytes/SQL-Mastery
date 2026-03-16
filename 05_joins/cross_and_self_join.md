# CROSS JOIN and SELF JOIN — Every Combination and Self-Reference

## The Analogies

**CROSS JOIN — Every combination of pizza size × topping:**

Think of a pizza shop's menu board. They have 3 sizes (Small, Medium, Large) and 4 toppings (Pepperoni, Mushroom, Olive, Jalapeño). To print every possible pizza option, you pair each size with every topping: 3 × 4 = **12 combinations**. No pairing is skipped, no matching condition is needed — it's a pure cartesian product.

**SELF JOIN — An org chart where a table talks to itself:**

You have an `employees` table where each row has a `manager_id` column that points back to another row in the **same table**. To show "employee name → manager name", you need to join `employees` to `employees`. You use two aliases — one for the employee, one for the manager — so the database treats them as two separate logical tables.

---

## CROSS JOIN — The Cartesian Product

A CROSS JOIN produces one output row for **every combination** of rows from the two input tables. There is no `ON` clause because there's no matching condition — every left row is paired with every right row.

```
sizes (3 rows)     toppings (4 rows)
───────────────    ─────────────────
Small              Pepperoni
Medium             Mushroom
Large              Olive
                   Jalapeño

CROSS JOIN produces 3 × 4 = 12 rows:

Small  × Pepperoni    Medium × Pepperoni    Large × Pepperoni
Small  × Mushroom     Medium × Mushroom     Large × Mushroom
Small  × Olive        Medium × Olive        Large × Olive
Small  × Jalapeño     Medium × Jalapeño     Large × Jalapeño
```

The cartesian product grows **multiplicatively** — 100 rows × 100 rows = 10,000 rows. Accidentally running a CROSS JOIN on large tables is a classic way to cripple a database.

---

## CROSS JOIN Syntax

```sql
CREATE TABLE sizes   (name TEXT);
CREATE TABLE toppings(name TEXT);

INSERT INTO sizes    VALUES ('Small'), ('Medium'), ('Large');
INSERT INTO toppings VALUES ('Pepperoni'), ('Mushroom'), ('Olive'), ('Jalapeño');

-- Explicit CROSS JOIN syntax
SELECT
    s.name AS size,
    t.name AS topping,
    CONCAT(s.name, ' + ', t.name) AS menu_item
FROM sizes    AS s
CROSS JOIN toppings AS t
ORDER BY s.name, t.name;

-- Old implicit syntax (comma-separated FROM — avoid this, it's error-prone)
-- SELECT s.name, t.name FROM sizes s, toppings t;
```

**Result (12 rows):**

```
  size   |  topping  |        menu_item
---------+-----------+-------------------------
 Large   | Jalapeño  | Large + Jalapeño
 Large   | Mushroom  | Large + Mushroom
 Large   | Olive     | Large + Olive
 Large   | Pepperoni | Large + Pepperoni
 Medium  | Jalapeño  | Medium + Jalapeño
 ...     | ...       | ...
 Small   | Pepperoni | Small + Pepperoni
```

---

## Real-World CROSS JOIN Use Cases

**1. Generate test data — pair every product with every store:**

```sql
SELECT
    p.id   AS product_id,
    p.name AS product_name,
    s.id   AS store_id,
    s.city AS store_city,
    0      AS stock_count    -- seed with zero, update later
FROM products AS p
CROSS JOIN stores AS s;
```

**2. Build a date-range scaffold for a reporting grid:**

```sql
-- Every month in 2024 × every product category
SELECT
    gs.month_start,
    c.category_name
FROM generate_series(
        '2024-01-01'::date,
        '2024-12-01'::date,
        '1 month'::interval
     ) AS gs(month_start)
CROSS JOIN categories AS c
ORDER BY gs.month_start, c.category_name;
```

This gives you a complete grid with no gaps — you can then LEFT JOIN actual sales data onto it so months with zero sales still appear as rows (rather than being silently absent).

---

## The Explosion Warning

```
Rows in A  × Rows in B  = Output rows
─────────────────────────────────────
        10 ×         10 =         100
     1,000 ×      1,000 =   1,000,000
    10,000 ×     10,000 = 100,000,000  ← will probably crash your session
```

Always know your row counts before writing a CROSS JOIN in production. If you accidentally omit an `ON` clause in a regular JOIN, some databases (especially older MySQL configurations) silently produce a cartesian product.

---

## SELF JOIN — Joining a Table to Itself

A SELF JOIN is not a special join type — it is just a regular JOIN (usually INNER or LEFT) where **both sides reference the same table**, distinguished by different aliases.

**The classic use case: employee hierarchy**

```sql
CREATE TABLE employees (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    title      TEXT,
    manager_id INT REFERENCES employees(id)
);

INSERT INTO employees (id, name, title, manager_id) VALUES
    (1, 'Sarah Chen',    'CEO',              NULL),
    (2, 'James Okafor',  'VP Engineering',      1),
    (3, 'Priya Sharma',  'VP Marketing',         1),
    (4, 'Luca Rossi',    'Senior Engineer',      2),
    (5, 'Anna Kowalski', 'Engineer',             2),
    (6, 'Mohamed Ali',   'Marketing Lead',       3);
```

**SELF JOIN to show employee → manager pairs:**

```sql
SELECT
    e.id            AS emp_id,
    e.name          AS employee,
    e.title         AS emp_title,
    m.name          AS manager_name,
    m.title         AS manager_title
FROM employees AS e          -- "e" = the employee side
LEFT JOIN employees AS m     -- "m" = the manager side (same table, different alias)
    ON e.manager_id = m.id
ORDER BY e.id;
```

**Result:**

```
 emp_id | employee      | emp_title        | manager_name  | manager_title
--------+---------------+------------------+---------------+----------------
      1 | Sarah Chen    | CEO              | NULL          | NULL
      2 | James Okafor  | VP Engineering   | Sarah Chen    | CEO
      3 | Priya Sharma  | VP Marketing     | Sarah Chen    | CEO
      4 | Luca Rossi    | Senior Engineer  | James Okafor  | VP Engineering
      5 | Anna Kowalski | Engineer         | James Okafor  | VP Engineering
      6 | Mohamed Ali   | Marketing Lead   | Priya Sharma  | VP Marketing
```

Note: LEFT JOIN preserves Sarah Chen even though her `manager_id` is NULL.

---

## How the Alias Trick Works

```
Physical table in memory:  employees
                                │
          ┌─────────────────────┴─────────────────────┐
          │                                           │
    alias "e"                                   alias "m"
  (read as: employee)                       (read as: manager)
          │                                           │
     e.manager_id ──────────────── = ────────── m.id
```

The database treats `e` and `m` as if they were two separate tables that happen to contain the same data. The JOIN condition links `e.manager_id` to `m.id` — the employee's manager ID matches the manager's own ID.

---

## SELF JOIN for Comparing Rows Within a Table

Self joins aren't just for hierarchies. Use them anytime you need to compare rows within the same table.

```sql
-- Find pairs of products in the same category with similar prices (within £10)
SELECT
    a.name  AS product_a,
    b.name  AS product_b,
    a.price AS price_a,
    b.price AS price_b,
    ABS(a.price - b.price) AS price_diff
FROM products AS a
INNER JOIN products AS b
    ON  a.category = b.category
    AND a.id < b.id                     -- avoid (A,B) and (B,A) duplicates
    AND ABS(a.price - b.price) <= 10
ORDER BY price_diff;
```

The `a.id < b.id` condition is a standard technique to avoid producing both `(Laptop A, Laptop B)` and `(Laptop B, Laptop A)` as separate rows.

---

## Summary

```
┌────────────────────────────────────────────────────────────────────┐
│                 CROSS JOIN and SELF JOIN CHEAT SHEET               │
├───────────────────────┬────────────────────────────────────────────┤
│ CROSS JOIN            │ Every row from A × every row from B        │
│ CROSS JOIN syntax     │ FROM a CROSS JOIN b  (no ON clause)        │
│ Row count warning     │ |A| × |B| rows — grows fast                │
│ Good CROSS JOIN uses  │ Scaffold grids, generate combinations      │
│ SELF JOIN             │ Regular JOIN with same table, 2 aliases    │
│ Typical uses          │ Hierarchy, row comparison, adjacency       │
│ Prevent duplicates    │ Use a.id < b.id when pairing rows          │
│ Hierarchy tip         │ Use LEFT JOIN so root nodes (NULL mgr)     │
│                       │ are not dropped                            │
└───────────────────────┴────────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← LEFT / RIGHT / OUTER JOIN](./left_right_outer_join.md) &nbsp;|&nbsp; **Next:** [JOIN Patterns →](./join_patterns.md)

**Related Topics:** [INNER JOIN](./inner_join.md) · [LEFT/RIGHT/OUTER JOIN](./left_right_outer_join.md) · [JOIN Patterns](./join_patterns.md)
