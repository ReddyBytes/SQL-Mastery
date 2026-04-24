# Normalization

## The Messy Spreadsheet Problem

Imagine a sales team tracking orders in a shared spreadsheet. Every time someone places an order, they add a row with everything they know: the customer's name, their city, their email, the product, the product category, the supplier, and the price.

After six months the spreadsheet has 3,000 rows and looks like this:

```
order_id | customer   | city    | email               | product          | category    | supplier       | price
---------|------------|---------|---------------------|------------------|-------------|----------------|-------
1001     | Alice Wang | London  | alice@example.com   | Wireless Kbd     | Electronics | TechSupply Ltd | 79.99
1002     | Bob Singh  | Leeds   | bob@example.com     | USB-C Hub        | Electronics | TechSupply Ltd | 45.00
1003     | Alice Wang | London  | alice@example.com   | Monitor Stand    | Furniture   | OfficePro      | 129.00
1004     | Bob Singh  | Leeds   | bob@example.com     | Wireless Kbd     | Electronics | TechSupply Ltd | 79.99
1005     | Alice Wang | Londn   | alice@example.com   | Laptop Sleeve    | Accessories | TechSupply Ltd | 34.50
```

Spot the problem? Row 1005 has "Londn" instead of "London." Someone made a typo. Now every report about London is wrong. And when Alice Wang moves to Manchester, someone has to update every single row where she appears — and probably misses one.

This is the problem normalization solves: **one source of truth, no redundancy, no anomalies.**

---

## What is Normalization?

Normalization is the process of structuring a relational database to reduce data redundancy and improve data integrity. It works through a series of "normal forms" — each one a stricter set of rules than the last.

In practice, most production databases aim for **Third Normal Form (3NF)**.

---

## The Messy Starting Table

Let us work with this single denormalized table and progressively fix it:

```sql
CREATE TABLE orders_flat (
    order_id         INT,
    order_date       DATE,
    customer_id      INT,
    customer_name    VARCHAR(100),
    customer_email   VARCHAR(255),
    customer_city    VARCHAR(100),
    product_id       INT,
    product_name     VARCHAR(200),
    category_name    VARCHAR(100),
    supplier_name    VARCHAR(100),
    unit_price       DECIMAL(10, 2),
    quantity         INT
);
```

Problems with this table:
- Customer data is repeated on every order they place
- Product and category data is repeated on every order containing that product
- One typo in `customer_city` corrupts all reports for that customer
- Deleting an order might destroy the only record of a supplier
- Updating a product price means updating every row that references that product

---

## First Normal Form (1NF)

**Rule:** Every column contains atomic (indivisible) values. No repeating groups or arrays in a column. Each row is uniquely identifiable.

### Violation Example

```
order_id | customer  | products
---------|-----------|----------------------------------
1001     | Alice     | Wireless Kbd, USB-C Hub          <- NOT atomic
1002     | Bob       | Monitor Stand                    <- OK
```

The `products` column stores a list. You cannot meaningfully query "find all orders containing a Wireless Kbd" without string parsing.

### Fix — Split into Separate Rows

```
order_id | customer | product
---------|----------|---------------
1001     | Alice    | Wireless Kbd
1001     | Alice    | USB-C Hub
1002     | Bob      | Monitor Stand
```

Our `orders_flat` table already passes 1NF — each cell holds one value. But we still have redundancy problems.

---

## Second Normal Form (2NF)

**Rule:** The table must be in 1NF **and** every non-key column must depend on the **whole** primary key, not just part of it.

This only applies to tables with **composite primary keys**.

### Violation Example

Suppose our flat table used `(order_id, product_id)` as a composite PK:

```
PK: (order_id, product_id)

order_id | product_id | order_date | customer_name | product_name | quantity
---------|------------|------------|---------------|--------------|--------
1001     | P10        | 2024-01-05 | Alice Wang    | Wireless Kbd | 1
1001     | P11        | 2024-01-05 | Alice Wang    | USB-C Hub    | 2
1002     | P10        | 2024-01-07 | Bob Singh     | Wireless Kbd | 1
```

- `order_date` and `customer_name` depend only on `order_id` — not on `product_id`
- `product_name` depends only on `product_id` — not on `order_id`

These are **partial dependencies** — a 2NF violation.

### Fix — Split into Tables by Dependency

```sql
-- order_date depends on order_id alone -> goes in orders table
CREATE TABLE orders (
    order_id    INT     PRIMARY KEY,
    customer_id INT     NOT NULL,
    order_date  DATE    NOT NULL
);

-- product_name depends on product_id alone -> goes in products table
CREATE TABLE products (
    product_id   INT            PRIMARY KEY,
    product_name VARCHAR(200)   NOT NULL,
    unit_price   DECIMAL(10,2)  NOT NULL
);

-- quantity depends on BOTH order_id AND product_id -> stays in junction table
CREATE TABLE order_items (
    order_id   INT NOT NULL REFERENCES orders(order_id),
    product_id INT NOT NULL REFERENCES products(product_id),
    quantity   INT NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

---

## Third Normal Form (3NF)

**Rule:** The table must be in 2NF **and** no non-key column should depend on another non-key column (no **transitive dependencies**).

### Violation Example

```
customers table:
customer_id | customer_name | city       | country
------------|---------------|------------|--------
101         | Alice Wang    | London     | UK
102         | Bob Singh     | Leeds      | UK
103         | Carol Müller  | Berlin     | Germany
```

`country` depends on `city`, not directly on `customer_id`. If we have 1,000 customers in London, "UK" is stored 1,000 times. If London gets reclassified (hypothetically), we must update 1,000 rows.

`city -> country` is a transitive dependency.

### Fix — Extract the Dependency

```sql
CREATE TABLE cities (
    city_id   SERIAL       PRIMARY KEY,
    city_name VARCHAR(100) NOT NULL,
    country   VARCHAR(100) NOT NULL
);

CREATE TABLE customers (
    customer_id   SERIAL       PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    email         VARCHAR(255) NOT NULL UNIQUE,
    city_id       INT          REFERENCES cities(city_id)
);
```

Now "UK" is stored once in the `cities` table. Change it there and every customer in London automatically reflects the change.

---

## The Fully Normalized Schema

Applying all three normal forms to our original flat table:

```
orders_flat  (before)
     |
     v

customers ----+
cities    --->+
              |
orders    ----+----> order_items ----> products
                                       |
                                       v
                                   categories
                                   suppliers
```

```sql
CREATE TABLE categories (
    category_id   SERIAL       PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE suppliers (
    supplier_id   SERIAL       PRIMARY KEY,
    supplier_name VARCHAR(200) NOT NULL
);

CREATE TABLE products (
    product_id    SERIAL          PRIMARY KEY,
    product_name  VARCHAR(200)    NOT NULL,
    category_id   INT             REFERENCES categories(category_id),
    supplier_id   INT             REFERENCES suppliers(supplier_id),
    unit_price    DECIMAL(10,2)   NOT NULL CHECK (unit_price >= 0)
);

CREATE TABLE cities (
    city_id       SERIAL          PRIMARY KEY,
    city_name     VARCHAR(100)    NOT NULL,
    country       VARCHAR(100)    NOT NULL
);

CREATE TABLE customers (
    customer_id   SERIAL          PRIMARY KEY,
    customer_name VARCHAR(100)    NOT NULL,
    email         VARCHAR(255)    NOT NULL UNIQUE,
    city_id       INT             REFERENCES cities(city_id)
);

CREATE TABLE orders (
    order_id      SERIAL          PRIMARY KEY,
    customer_id   INT             NOT NULL REFERENCES customers(customer_id),
    order_date    DATE            NOT NULL DEFAULT CURRENT_DATE
);

CREATE TABLE order_items (
    order_id      INT             NOT NULL REFERENCES orders(order_id),
    product_id    INT             NOT NULL REFERENCES products(product_id),
    quantity      INT             NOT NULL CHECK (quantity > 0),
    unit_price    DECIMAL(10,2)   NOT NULL,  -- snapshot of price at time of order
    PRIMARY KEY (order_id, product_id)
);
```

"Londn" can never happen again — the city name lives in exactly one row of one table.

---

## When to Denormalize

Normalization is the right default. But it is not a universal law. Certain workloads benefit from intentional denormalization.

### Reasons to Denormalize

- **Read-heavy reporting:** a dashboard that joins 8 tables on every page load is slow. A pre-joined summary table is faster.
- **Data warehouses:** OLAP systems (Redshift, BigQuery, Snowflake) are often designed with wide, flat tables for analytical queries.
- **Caching frequently-joined data:** storing a `customer_city` column directly on `orders` saves a join on every order query.

### The Practical Rule

```
Normalize for writes: fewer places to update, no anomalies, strong integrity.
Denormalize for reads: fewer joins, faster queries, simpler SELECT statements.
```

A common pattern:
1. Keep your transactional (OLTP) database normalized to 3NF.
2. Build a separate reporting schema (or materialized views) that denormalizes for analytical queries.

```sql
-- Denormalized reporting view (not a table — refreshed on demand)
CREATE MATERIALIZED VIEW order_summary AS
SELECT
    o.order_id,
    o.order_date,
    c.customer_name,
    ct.city_name,
    ct.country,
    p.product_name,
    cat.category_name,
    s.supplier_name,
    oi.quantity,
    oi.unit_price,
    (oi.quantity * oi.unit_price) AS line_total
FROM orders o
JOIN customers c    ON o.customer_id  = c.customer_id
JOIN cities ct      ON c.city_id      = ct.city_id
JOIN order_items oi ON o.order_id     = oi.order_id
JOIN products p     ON oi.product_id  = p.product_id
JOIN categories cat ON p.category_id  = cat.category_id
JOIN suppliers s    ON p.supplier_id  = s.supplier_id;
```

---

## Summary

```
+------+------------------------------------------------------------------------+
| Form | Rule                                                                   |
+------+------------------------------------------------------------------------+
| 1NF  | Atomic values only. No repeating groups. Each row uniquely identified. |
| 2NF  | 1NF + no partial dependencies (all columns depend on the full PK).     |
| 3NF  | 2NF + no transitive dependencies (non-key depends only on PK).         |
+------+------------------------------------------------------------------------+

Anomalies prevented by normalization:
  Insert anomaly  -- cannot record data without inserting an unrelated row
  Update anomaly  -- changing one fact requires updating many rows
  Delete anomaly  -- deleting one row accidentally destroys other data

When to denormalize:
  - Read-heavy reporting / analytics
  - Pre-computed aggregates for dashboards
  - Materialized views for complex joins
  - Data warehouses (star/snowflake schemas)

Golden rule:
  Normalize first. Denormalize only when you can measure the performance benefit.
```

---

**[Back to README](../README.md)**

**Prev:** [← Data Types](./data_types.md) &nbsp;|&nbsp; **Next:** [Indexes →](./indexes.md)

**Related Topics:** [Tables and Constraints](./tables_and_constraints.md) · [Data Types](./data_types.md) · [Indexes](./indexes.md)

---

## 📝 Practice Questions

- 📝 [Q34 · normalization-1nf](../sql_practice_questions_100.md#q34--interview--normalization-1nf)
- 📝 [Q35 · normalization-2nf](../sql_practice_questions_100.md#q35--interview--normalization-2nf)
- 📝 [Q36 · normalization-3nf](../sql_practice_questions_100.md#q36--interview--normalization-3nf)
- 📝 [Q37 · denormalization](../sql_practice_questions_100.md#q37--design--denormalization)
- 📝 [Q82 · compare-norm-denorm](../sql_practice_questions_100.md#q82--interview--compare-norm-denorm)

