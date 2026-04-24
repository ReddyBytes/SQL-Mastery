# INSERT — Adding Data to Your Tables

## The Filing Cabinet Analogy

Picture a large office filing cabinet. Each drawer is a table, and each folder inside is a row of data. When a new employee joins the company, someone has to physically walk over and file their paperwork — one folder at a time, or if it's a busy hire week, they might bring a whole stack and file them all at once.

`INSERT` is that action. It's how data gets into your database. You can add one row carefully, or push in hundreds at once. Either way, the cabinet rules apply: every folder must go into the right drawer, with the right labels in the right order.

---

## Basic INSERT Syntax

```sql
INSERT INTO table_name (column1, column2, column3)
VALUES (value1, value2, value3);
```

Always list your column names explicitly. It protects you if the table schema changes later.

```sql
-- Adding a new user to the users table
INSERT INTO users (name, email, created_at)
VALUES ('Sarah Connor', 'sarah@example.com', NOW());
```

### What happens without column names?

```sql
-- Fragile: depends on exact column order in the table definition
INSERT INTO users
VALUES (42, 'Sarah Connor', 'sarah@example.com', NOW(), TRUE);
```

If someone adds a column between `id` and `name` next month, this insert silently puts data in the wrong place — or breaks entirely. Always name your columns.

---

## Inserting Multiple Rows at Once

Instead of running 100 separate INSERT statements, send them all in one batch. The database commits them together, which is dramatically faster.

```sql
INSERT INTO products (name, category, price, stock_qty)
VALUES
  ('Wireless Mouse',     'Electronics', 29.99,  150),
  ('Mechanical Keyboard','Electronics', 89.99,   75),
  ('Desk Lamp',          'Office',      19.99,  200),
  ('Notebook (A5)',      'Stationery',   4.99, 1000),
  ('USB-C Hub',          'Electronics', 49.99,   60);
```

```
Batch INSERT performance vs single inserts:

Single (100 rows):  100 round-trips × network latency = slow
Batch  (100 rows):  1  round-trip  × network latency = fast

Rule of thumb: batch anything over ~10 rows.
```

---

## INSERT INTO ... SELECT (Copy From Another Table)

You can populate a table directly from a query — no VALUES clause needed. This is perfect for archiving, migrating, or seeding data.

```sql
-- Archive orders older than 2 years into an orders_archive table
INSERT INTO orders_archive (order_id, user_id, total_amount, created_at)
SELECT order_id, user_id, total_amount, created_at
FROM   orders
WHERE  created_at < NOW() - INTERVAL '2 years';
```

The SELECT can be as complex as you need — JOINs, filters, calculated columns, all fine.

```sql
-- Seed a reporting table with aggregated monthly sales
INSERT INTO monthly_sales_summary (year, month, total_revenue, order_count)
SELECT
    EXTRACT(YEAR  FROM created_at) AS year,
    EXTRACT(MONTH FROM created_at) AS month,
    SUM(total_amount)              AS total_revenue,
    COUNT(*)                       AS order_count
FROM   orders
WHERE  status = 'completed'
GROUP  BY 1, 2;
```

---

## ON CONFLICT — Upsert in PostgreSQL

Real production systems often face a dilemma: "Insert this row, but if it already exists, update it instead." This is called an **upsert** (update + insert).

PostgreSQL handles this elegantly with `ON CONFLICT`:

```
Flow:
  INSERT row
       |
       v
  Does a row with this PK / unique key already exist?
     NO  --> insert normally
     YES --> do what ON CONFLICT says
```

### Do Nothing on Conflict

```sql
-- Silently skip if a user with this email already exists
INSERT INTO users (email, name, created_at)
VALUES ('alice@example.com', 'Alice Zhang', NOW())
ON CONFLICT (email) DO NOTHING;
```

### Update on Conflict

```sql
-- Update the price if a product SKU already exists
INSERT INTO products (sku, name, price, updated_at)
VALUES ('WLESS-MSE-001', 'Wireless Mouse Pro', 34.99, NOW())
ON CONFLICT (sku)
DO UPDATE SET
    price      = EXCLUDED.price,
    updated_at = EXCLUDED.updated_at;
```

`EXCLUDED` refers to the row that was *attempted* to be inserted — the one that triggered the conflict. It's PostgreSQL's way of saying "the row you tried to add."

> **MySQL equivalent:** `INSERT ... ON DUPLICATE KEY UPDATE`
> ```sql
> INSERT INTO products (sku, name, price)
> VALUES ('WLESS-MSE-001', 'Wireless Mouse Pro', 34.99)
> ON DUPLICATE KEY UPDATE price = VALUES(price);
> ```

---

## RETURNING — Get Back What You Just Inserted

In PostgreSQL, you can chain `RETURNING` onto an INSERT to immediately get back column values from the new row — most commonly the auto-generated primary key.

```sql
-- Insert a new order and capture the generated order_id
INSERT INTO orders (user_id, status, total_amount, created_at)
VALUES (1042, 'pending', 129.97, NOW())
RETURNING order_id;
```

```
Result:
 order_id
----------
    50231
```

This is invaluable when you need the new ID to insert related rows (like `order_items`) in the same operation.

```sql
-- Full pattern: insert order, get ID, use it for line items
WITH new_order AS (
    INSERT INTO orders (user_id, status, total_amount, created_at)
    VALUES (1042, 'pending', 129.97, NOW())
    RETURNING order_id
)
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT no.order_id, 88, 2, 49.99
FROM   new_order no;
```

---

## Common Mistakes

### 1. Wrong Column Order

```sql
-- Table: users(id, name, email, created_at)

-- BUG: email and name are swapped
INSERT INTO users (id, email, name, created_at)
VALUES (1, 'Alice Zhang', 'alice@example.com', NOW());
-- 'Alice Zhang' goes into the email column!
```

Always double-check your column list matches your values list left-to-right.

### 2. Inserting Duplicate Primary Keys

```sql
-- Will fail with: ERROR: duplicate key value violates unique constraint
INSERT INTO users (id, name, email)
VALUES (1, 'Bob Smith', 'bob@example.com');
-- If user id=1 already exists
```

Use `ON CONFLICT DO NOTHING` or `ON CONFLICT DO UPDATE` to handle this gracefully.

### 3. NULL in a NOT NULL Column

```sql
-- Will fail: ERROR: null value in column "email" violates not-null constraint
INSERT INTO users (name, email)
VALUES ('Charlie Brown', NULL);
```

Validate your data before inserting, or set a sensible `DEFAULT` in the table definition.

### 4. Forgetting to Commit in a Transaction

If you're running inserts inside a transaction block (`BEGIN`), the data is invisible to other sessions until you `COMMIT`. Don't close your session without committing — the changes will be rolled back silently.

---

## Quick Reference

```
+---------------------------------------+------------------------------------------+
| Task                                  | Syntax                                   |
+---------------------------------------+------------------------------------------+
| Insert one row                        | INSERT INTO t (col1, col2) VALUES (...)  |
| Insert multiple rows                  | INSERT INTO t (...) VALUES (...),(...)   |
| Copy rows from another table          | INSERT INTO t (...) SELECT ... FROM s   |
| Upsert (insert or update on conflict) | ... ON CONFLICT (col) DO UPDATE SET ... |
| Skip if duplicate exists              | ... ON CONFLICT (col) DO NOTHING        |
| Return generated ID after insert      | INSERT INTO t (...) VALUES (...) RETURNING id |
+---------------------------------------+------------------------------------------+
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← String and Date Functions](../06_advanced_queries/string_and_date_functions.md) &nbsp;|&nbsp; **Next:** [UPDATE and DELETE →](./update_and_delete.md)

**Related Topics:** [UPDATE and DELETE](./update_and_delete.md) · [Transactions](./transactions.md)

---

## 📝 Practice Questions

- 📝 [Q56 · insert-basics](../sql_practice_questions_100.md#q56--normal--insert-basics)
- 📝 [Q57 · upsert-on-conflict](../sql_practice_questions_100.md#q57--design--upsert-on-conflict)

