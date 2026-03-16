# Tables and Constraints

## The Government Form Analogy

Have you ever filled out a government form — a passport application, a tax return, a driving licence renewal? Each field has a specific type (letters only, numbers only, a date), some fields are mandatory (you cannot submit without them), some must be unique across all applicants (your national ID number), and some fields refer to other forms you have already submitted (your previous passport number).

A database table is exactly like that form, but enforced automatically and at scale. Constraints are the form's rules made permanent and machine-checked — the database refuses to accept invalid data rather than waiting for a human reviewer to catch the mistake three weeks later.

---

## CREATE TABLE — Defining a Table

The basic syntax:

```sql
CREATE TABLE table_name (
    column_name  data_type  [constraints],
    column_name  data_type  [constraints],
    ...
    [table-level constraints]
);
```

Let us build a real e-commerce schema step by step.

### Users Table

```sql
CREATE TABLE users (
    user_id    SERIAL          PRIMARY KEY,
    email      VARCHAR(255)    NOT NULL UNIQUE,
    username   VARCHAR(50)     NOT NULL UNIQUE,
    full_name  VARCHAR(100)    NOT NULL,
    created_at TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    is_active  BOOLEAN         NOT NULL DEFAULT TRUE
);
```

### Products Table

```sql
CREATE TABLE products (
    product_id   SERIAL           PRIMARY KEY,
    sku          VARCHAR(50)      NOT NULL UNIQUE,
    name         VARCHAR(200)     NOT NULL,
    description  TEXT,
    price        DECIMAL(10, 2)   NOT NULL CHECK (price >= 0),
    stock_qty    INT              NOT NULL DEFAULT 0 CHECK (stock_qty >= 0),
    category     VARCHAR(100),
    created_at   TIMESTAMPTZ      NOT NULL DEFAULT NOW()
);
```

### Orders Table

```sql
CREATE TABLE orders (
    order_id    SERIAL          PRIMARY KEY,
    user_id     INT             NOT NULL REFERENCES users(user_id),
    status      VARCHAR(20)     NOT NULL DEFAULT 'pending'
                                CHECK (status IN ('pending','processing','shipped','delivered','refunded')),
    total       DECIMAL(10, 2)  NOT NULL DEFAULT 0.00,
    created_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);
```

### Order Items Table

```sql
CREATE TABLE order_items (
    item_id     SERIAL           PRIMARY KEY,
    order_id    INT              NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id  INT              NOT NULL REFERENCES products(product_id),
    quantity    INT              NOT NULL CHECK (quantity > 0),
    unit_price  DECIMAL(10, 2)  NOT NULL CHECK (unit_price >= 0),

    -- Composite unique: same product cannot appear twice in one order
    UNIQUE (order_id, product_id)
);
```

### Schema Diagram

```
users                products
+---------+          +------------+
| user_id |<----+    | product_id |<----+
| email   |     |    | sku        |     |
| username|     |    | name       |     |
| ...     |     |    | price      |     |
+---------+     |    | ...        |     |
                |    +------------+     |
            orders                      |
            +-----------+               |
            | order_id  |<---------+    |
            | user_id   |--+       |    |
            | status    |  |  order_items
            | total     |  |  +------------+
            | ...       |  |  | item_id    |
            +-----------+  |  | order_id   |--+
                           |  | product_id |------+
                           |  | quantity   |
                           |  | unit_price |
                           |  +------------+
                           |
                    (FK to users)
```

---

## Constraint Types — Reference Guide

### PRIMARY KEY

Uniquely identifies each row. Implies NOT NULL + UNIQUE. Every table should have one.

```sql
-- Single column (most common)
user_id SERIAL PRIMARY KEY

-- Composite primary key (table-level syntax)
PRIMARY KEY (order_id, product_id)
```

### FOREIGN KEY (Referential Integrity)

Links a column to the primary key of another table. The database refuses to insert a row that references a non-existent parent.

```sql
-- Inline
user_id INT NOT NULL REFERENCES users(user_id)

-- Table-level (same effect)
FOREIGN KEY (user_id) REFERENCES users(user_id)
```

### NOT NULL

The column must have a value — NULL is rejected.

```sql
email VARCHAR(255) NOT NULL
```

### UNIQUE

No two rows may have the same value in this column (NULLs are treated specially — most databases allow multiple NULL values in a UNIQUE column).

```sql
email VARCHAR(255) UNIQUE
```

### CHECK

A Boolean expression that every new or updated row must satisfy.

```sql
price DECIMAL(10, 2) CHECK (price >= 0)
age   INT             CHECK (age BETWEEN 0 AND 150)
```

### DEFAULT

Automatically fills a value if one is not supplied at insert time.

```sql
is_active  BOOLEAN     DEFAULT TRUE
created_at TIMESTAMPTZ DEFAULT NOW()
```

---

## Referential Integrity — What Happens When You Delete a Parent Row?

This is one of the most important things to understand in relational databases. If `orders.user_id` references `users.user_id`, what happens if you try to delete a user who has orders?

You control this with the `ON DELETE` clause on the foreign key:

```sql
-- Option 1: RESTRICT (default in most databases)
-- Block the delete if child rows exist
user_id INT REFERENCES users(user_id) ON DELETE RESTRICT

-- Option 2: CASCADE
-- Delete the child rows automatically
order_id INT REFERENCES orders(order_id) ON DELETE CASCADE

-- Option 3: SET NULL
-- Set the foreign key column to NULL in child rows
user_id INT REFERENCES users(user_id) ON DELETE SET NULL

-- Option 4: SET DEFAULT
-- Set the foreign key column to its DEFAULT value
user_id INT REFERENCES users(user_id) ON DELETE SET DEFAULT
```

### When to Use Each

```
+------------+------------------------------------------------------------+
| Option     | Use when...                                                |
+------------+------------------------------------------------------------+
| RESTRICT   | Parent deletion should be prevented (safest default)       |
| CASCADE    | Child records have no meaning without the parent           |
|            | (e.g., order_items without orders)                        |
| SET NULL   | Child records can exist independently (orphaned is OK)     |
| SET DEFAULT| Child should fall back to a default parent                 |
+------------+------------------------------------------------------------+
```

Example: A user deletes their account. Their orders contain purchase history you want to keep for accounting purposes — use `SET NULL` on `orders.user_id`. But the order items only make sense as part of an order — use `CASCADE` on `order_items.order_id`.

---

## ALTER TABLE — Modifying an Existing Table

You will often need to change a table after it has been created and populated.

```sql
-- Add a new column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Add a column with a default (safe on large tables)
ALTER TABLE products ADD COLUMN is_featured BOOLEAN NOT NULL DEFAULT FALSE;

-- Drop a column
ALTER TABLE users DROP COLUMN phone;

-- Rename a column
ALTER TABLE users RENAME COLUMN full_name TO display_name;

-- Change a column's data type
ALTER TABLE products ALTER COLUMN sku TYPE VARCHAR(100);

-- Add a constraint after the fact
ALTER TABLE products ADD CONSTRAINT chk_price CHECK (price > 0);

-- Drop a constraint
ALTER TABLE products DROP CONSTRAINT chk_price;

-- Add a foreign key after the fact
ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(user_id);
```

> **Production warning:** `ALTER TABLE` on large tables can lock the table for writes. In PostgreSQL, adding a column with a constant DEFAULT is instant; adding a NOT NULL column without a DEFAULT requires a full table rewrite. Plan schema changes carefully.

---

## DROP TABLE — Removing a Table

```sql
-- Remove the table and all its data
DROP TABLE order_items;

-- Only drop if it exists (avoids an error if it does not)
DROP TABLE IF EXISTS order_items;

-- Remove a table and all tables that reference it (dangerous!)
DROP TABLE orders CASCADE;
```

> **Warning:** `DROP TABLE` is irreversible without a backup. Use `IF EXISTS` in scripts to avoid errors during idempotent deploys.

---

## Summary

```
+----------------+-------------------------------------------------------+
| Constraint     | Meaning                                               |
+----------------+-------------------------------------------------------+
| PRIMARY KEY    | Unique, not-null row identifier                       |
| FOREIGN KEY    | Must match an existing value in another table         |
| NOT NULL       | Column must always have a value                       |
| UNIQUE         | No duplicate values in this column                   |
| CHECK(expr)    | Row must satisfy this Boolean condition               |
| DEFAULT value  | Used when INSERT does not supply a value              |
+----------------+-------------------------------------------------------+

ON DELETE actions (foreign key behaviour):
  RESTRICT   -- block deletion of parent (default)
  CASCADE    -- delete children automatically
  SET NULL   -- set FK column to NULL
  SET DEFAULT-- set FK column to its default value

ALTER TABLE quick reference:
  ADD COLUMN col type [constraint]
  DROP COLUMN col
  RENAME COLUMN old TO new
  ALTER COLUMN col TYPE new_type
  ADD CONSTRAINT name ...
  DROP CONSTRAINT name
```

---

**[Back to README](../README.md)**

**Prev:** [← Window Functions](../03_aggregation/window_functions.md) &nbsp;|&nbsp; **Next:** [Data Types →](./data_types.md)

**Related Topics:** [Data Types](./data_types.md) · [Normalization](./normalization.md) · [Indexes](./indexes.md)
