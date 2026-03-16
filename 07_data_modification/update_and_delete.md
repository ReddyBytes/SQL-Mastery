# UPDATE and DELETE — Correcting and Removing Records

## The Filing Cabinet Analogy

You're auditing the office filing cabinet. Two tasks come up:

1. An employee's address changed — you pull out their folder, cross out the old address, and write the new one. That's `UPDATE`.
2. A contract expired three years ago and legal says shred it — you pull the folder out and it's gone forever. That's `DELETE`.

Both actions are permanent (unless you're inside a transaction). Both need to be done with precision. The single most dangerous habit in SQL is running an UPDATE or DELETE without a WHERE clause — that's not fixing one folder, that's changing or destroying *every folder in the cabinet*.

---

## UPDATE — Changing Existing Data

### Basic Syntax

```sql
UPDATE table_name
SET    column1 = value1,
       column2 = value2
WHERE  condition;
```

### Simple Example: Fix a Typo in One Row

```sql
UPDATE users
SET    email = 'alice.zhang@example.com'
WHERE  user_id = 1042;
```

### Update Multiple Columns at Once

```sql
UPDATE products
SET    price      = 34.99,
       updated_at = NOW()
WHERE  sku = 'WLESS-MSE-001';
```

### Update a Whole Category (with care)

```sql
-- Apply a 10% price increase to all Electronics products
UPDATE products
SET    price      = price * 1.10,
       updated_at = NOW()
WHERE  category = 'Electronics';
```

```
Before:
  sku           | category    | price
  WLESS-MSE-001 | Electronics | 29.99
  USB-C-HUB-03  | Electronics | 49.99
  NOTEBOOK-A5   | Stationery  |  4.99

After UPDATE WHERE category = 'Electronics':
  sku           | category    | price
  WLESS-MSE-001 | Electronics | 32.99   <-- updated
  USB-C-HUB-03  | Electronics | 54.99   <-- updated
  NOTEBOOK-A5   | Stationery  |  4.99   <-- unchanged
```

---

## UPDATE with a JOIN (PostgreSQL Syntax)

Sometimes the value you want to set lives in another table. PostgreSQL handles this with `FROM`:

```sql
-- Apply the discount percentage from a promotions table to matching products
UPDATE products p
SET    price = p.price * (1 - pr.discount_pct / 100.0),
       updated_at = NOW()
FROM   promotions pr
WHERE  p.category = pr.category
  AND  pr.active = TRUE;
```

> **MySQL equivalent:** MySQL uses a different syntax — list both tables after `UPDATE`:
> ```sql
> UPDATE products p
> JOIN   promotions pr ON p.category = pr.category
> SET    p.price = p.price * (1 - pr.discount_pct / 100.0)
> WHERE  pr.active = 1;
> ```

---

## DELETE — Removing Rows

### Basic Syntax

```sql
DELETE FROM table_name
WHERE  condition;
```

### Delete a Single Row

```sql
DELETE FROM users
WHERE  user_id = 9999;
```

### Delete Based on a Condition

```sql
-- Remove all sessions older than 30 days
DELETE FROM user_sessions
WHERE  last_active < NOW() - INTERVAL '30 days';
```

### DELETE with a Subquery

```sql
-- Delete order items that belong to cancelled orders
DELETE FROM order_items
WHERE  order_id IN (
    SELECT order_id
    FROM   orders
    WHERE  status = 'cancelled'
);
```

---

## TRUNCATE vs DELETE

Both remove rows, but they work very differently:

```
+--------------------+---------------------------+---------------------------+
| Feature            | DELETE                    | TRUNCATE                  |
+--------------------+---------------------------+---------------------------+
| Removes all rows?  | Yes (without WHERE)       | Yes                       |
| Uses WHERE?        | Yes                       | No                        |
| Can be rolled back?| Yes (inside transaction)  | Yes (PostgreSQL); No (MySQL)|
| Fires triggers?    | Yes                       | No                        |
| Resets sequences?  | No                        | Yes (with RESTART IDENTITY)|
| Speed on big table | Slow (row-by-row logging) | Very fast                 |
+--------------------+---------------------------+---------------------------+
```

```sql
-- DELETE: slow but selective and trigger-aware
DELETE FROM event_logs WHERE created_at < '2022-01-01';

-- TRUNCATE: fast, resets auto-increment, no triggers
TRUNCATE TABLE event_logs;

-- PostgreSQL: truncate and reset the ID sequence
TRUNCATE TABLE event_logs RESTART IDENTITY;
```

Use `TRUNCATE` when you want to wipe a staging/temp table clean and start fresh. Use `DELETE` for selective removal in production.

---

## The Soft Delete Pattern

Deleting rows permanently is often the wrong call in production:

- You lose history and auditability
- Foreign keys may break
- Users might want to "undo" a deletion

The industry-standard solution is **soft delete**: add a `deleted_at` column (or `is_active` boolean) and mark records as deleted instead of removing them.

```sql
-- Schema: add deleted_at to users
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ DEFAULT NULL;

-- "Delete" a user: mark as deleted, don't remove the row
UPDATE users
SET    deleted_at = NOW()
WHERE  user_id = 5501;

-- In your application queries, always filter active users
SELECT * FROM users
WHERE  deleted_at IS NULL;

-- Create a view to make this automatic (covered in Views chapter)
CREATE VIEW active_users AS
SELECT * FROM users
WHERE  deleted_at IS NULL;
```

```
users table:
  user_id | name         | email              | deleted_at
  --------+--------------+--------------------+---------------------------
  1042    | Alice Zhang  | alice@example.com  | NULL          <-- active
  1043    | Bob Smith    | bob@example.com    | NULL          <-- active
  5501    | Old Account  | old@example.com    | 2024-11-15 09:23:00+00  <-- "deleted"
```

---

## Safety Rules — Always Follow These

### Rule 1: Test Your WHERE Clause as a SELECT First

Before running any UPDATE or DELETE, run the equivalent SELECT to see exactly which rows will be affected:

```sql
-- Step 1: See what you'd be deleting
SELECT user_id, name, email
FROM   users
WHERE  created_at < '2020-01-01' AND last_login IS NULL;

-- Step 2: If the results look right, proceed
DELETE FROM users
WHERE  created_at < '2020-01-01' AND last_login IS NULL;
```

### Rule 2: Wrap Dangerous Changes in a Transaction

```sql
BEGIN;

UPDATE orders
SET    status = 'refunded',
       updated_at = NOW()
WHERE  user_id = 1042 AND status = 'completed';

-- Check: how many rows were affected?
-- If it looks wrong, ROLLBACK instead of COMMIT

COMMIT;
-- or: ROLLBACK;
```

### Rule 3: Never Run UPDATE/DELETE Without WHERE in Production

```sql
-- CATASTROPHIC: updates every single row in the table
UPDATE users SET is_active = FALSE;

-- SAFE: only deactivates one user
UPDATE users SET is_active = FALSE WHERE user_id = 1042;
```

Most database IDEs have a "safe mode" that blocks WHERE-less updates. Turn it on.

---

## Real-World Scenario: Deactivating vs Deleting Users

```sql
-- BAD: hard delete — lose the user's history forever
DELETE FROM users WHERE user_id = 5501;

-- BETTER: soft delete — preserve audit trail
UPDATE users
SET    is_active   = FALSE,
       deleted_at  = NOW(),
       deleted_by  = 'admin_user_99'   -- who did this?
WHERE  user_id = 5501;

-- Their orders, reviews, and activity remain intact
-- You can reactivate them if needed
UPDATE users
SET    is_active  = TRUE,
       deleted_at = NULL,
       deleted_by = NULL
WHERE  user_id = 5501;
```

---

## Summary

```
+-------------------+------------------------------------------------------+
| Command           | Description                                          |
+-------------------+------------------------------------------------------+
| UPDATE ... SET    | Modify column values in existing rows                |
| UPDATE ... FROM   | Update using values from a joined table (PostgreSQL) |
| DELETE FROM       | Remove rows matching a condition                     |
| TRUNCATE          | Remove ALL rows quickly; resets sequences            |
| Soft delete       | Set deleted_at/is_active instead of hard deleting    |
+-------------------+------------------------------------------------------+

Safety checklist:
  [x] Always use WHERE with UPDATE and DELETE
  [x] Test your WHERE as a SELECT first
  [x] Use transactions for multi-step changes
  [x] Consider soft delete before hard delete
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← INSERT](./insert.md) &nbsp;|&nbsp; **Next:** [Transactions →](./transactions.md)

**Related Topics:** [INSERT](./insert.md) · [Transactions](./transactions.md)
