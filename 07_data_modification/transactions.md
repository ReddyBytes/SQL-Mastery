# Transactions — All or Nothing

## The Bank Transfer Analogy

Imagine transferring £500 from your savings account to your current account. Under the hood, the bank must do two things:

1. **Debit** savings: subtract £500
2. **Credit** current: add £500

What if the system crashes after step 1 but before step 2? Without protection, you've lost £500 — it left savings but never arrived in current. That's a nightmare.

A **transaction** wraps both steps in a single atomic unit. Either both happen, or neither happens. The bank's balance sheet stays consistent regardless of what goes wrong. This is the core promise SQL transactions make, and it applies to everything from bank transfers to e-commerce order placements.

---

## The Core Commands

```
BEGIN         -- start the transaction block
  ... SQL statements ...
COMMIT        -- save all changes permanently
  -- or --
ROLLBACK      -- undo all changes since BEGIN
```

```sql
-- Bank transfer: move £500 from account 101 to account 202
BEGIN;

UPDATE accounts
SET    balance = balance - 500
WHERE  account_id = 101;

UPDATE accounts
SET    balance = balance + 500
WHERE  account_id = 202;

COMMIT;  -- both changes are saved together
```

If anything goes wrong between `BEGIN` and `COMMIT`:

```sql
BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 101;

-- Something goes wrong (wrong account, insufficient funds, etc.)
ROLLBACK;  -- the debit is reversed, nothing changed
```

> **Note:** `START TRANSACTION` is the SQL standard equivalent of `BEGIN` and works in both PostgreSQL and MySQL.

---

## ACID Properties Explained Simply

ACID is the acronym for the four guarantees every transaction-supporting database must provide:

```
+-------------+------------------------------------------------------------+
| Property    | Plain English                                              |
+-------------+------------------------------------------------------------+
| Atomicity   | All steps succeed, or none of them do. No half-done work. |
| Consistency | The database moves from one valid state to another.        |
|             | Constraints (PK, FK, NOT NULL) are never violated.        |
| Isolation   | Concurrent transactions don't see each other's            |
|             | in-progress changes. Each one runs as if it's alone.      |
| Durability  | Once COMMIT is called, data survives crashes, power       |
|             | outages, and restarts. It's written to disk.              |
+-------------+------------------------------------------------------------+
```

A simple way to remember it:

```
A = All or nothing
C = Correct state guaranteed
I = Invisible to others until done
D = Durable — committed means committed
```

---

## SAVEPOINT — Partial Rollbacks

Sometimes you want to undo *part* of a transaction without losing everything. `SAVEPOINT` lets you plant a flag mid-transaction and roll back to it.

```sql
BEGIN;

INSERT INTO orders (user_id, status, total_amount, created_at)
VALUES (1042, 'pending', 259.97, NOW());

SAVEPOINT after_order;  -- plant a flag here

INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (50231, 888, 1, 259.97);  -- wrong product ID!

-- Oops — roll back only to the savepoint, order is preserved
ROLLBACK TO SAVEPOINT after_order;

-- Try again with the correct product ID
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (50231, 88, 1, 259.97);

COMMIT;
```

```
Timeline:
  BEGIN
    |--- INSERT order ---|--- SAVEPOINT after_order
                                   |--- INSERT wrong item ---> ROLLBACK TO SAVEPOINT
                                   |--- INSERT correct item
  COMMIT
```

---

## Real-World: Order Placement

Placing an order in an e-commerce system involves multiple tables. If any step fails, you don't want a half-placed order.

```sql
BEGIN;

-- Step 1: Create the order record
INSERT INTO orders (user_id, status, total_amount, created_at)
VALUES (1042, 'pending', 129.97, NOW())
RETURNING order_id;
-- Assume this returns order_id = 50231

-- Step 2: Add the line items
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES
  (50231, 12, 1, 89.99),
  (50231, 47, 2, 19.99);

-- Step 3: Deduct from inventory
UPDATE products
SET    stock_qty = stock_qty - 1
WHERE  product_id = 12;

UPDATE products
SET    stock_qty = stock_qty - 2
WHERE  product_id = 47;

-- Step 4: Log the payment attempt
INSERT INTO payment_log (order_id, amount, status, attempted_at)
VALUES (50231, 129.97, 'processing', NOW());

-- All 4 steps succeeded — lock it in
COMMIT;
```

If the inventory update in Step 3 fails (e.g., stock_qty would go negative due to a CHECK constraint), PostgreSQL automatically aborts the transaction. A `ROLLBACK` cleans up the order and order_items that were already inserted, leaving the database in its original consistent state.

---

## Isolation Levels

When multiple transactions run at the same time, they can interfere with each other in subtle ways. Isolation levels control how much interference is allowed.

### The Three Problems

```
Dirty Read:      Transaction A reads data that Transaction B has changed
                 but not yet committed. B then rolls back. A read garbage.

Non-Repeatable  Transaction A reads a row. B updates and commits it.
Read:           A reads the same row again — different value. Surprising.

Phantom Read:   Transaction A runs a COUNT. B inserts new rows and commits.
                A runs COUNT again — different number. Rows appeared from nowhere.
```

### The Four Isolation Levels

```
+------------------+-----------+-------------------+--------------+
| Isolation Level  | Dirty Read| Non-Repeatable Read| Phantom Read |
+------------------+-----------+-------------------+--------------+
| READ UNCOMMITTED | Possible  | Possible          | Possible     |
| READ COMMITTED   | Prevented | Possible          | Possible     |
| REPEATABLE READ  | Prevented | Prevented         | Possible (in |
|                  |           |                   | most DBs)    |
| SERIALIZABLE     | Prevented | Prevented         | Prevented    |
+------------------+-----------+-------------------+--------------+

PostgreSQL default: READ COMMITTED
```

### Setting the Isolation Level

```sql
-- For a single transaction
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SELECT SUM(balance) FROM accounts WHERE user_id = 1042;
-- ...other work...
-- This SUM will return the same value even if other transactions
-- commit changes to these rows during our transaction

COMMIT;
```

```sql
-- Use SERIALIZABLE for financial calculations that must be exact
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Check balance before transfer
SELECT balance FROM accounts WHERE account_id = 101;
-- Transfer logic...
COMMIT;
```

**When to use each:**

- **READ COMMITTED** (default): Fine for most OLTP workloads. Each statement sees the latest committed data.
- **REPEATABLE READ**: Use when a single transaction must read the same data multiple times and get consistent results (e.g., generating a report mid-transaction).
- **SERIALIZABLE**: Use for strict financial operations where phantom reads could cause double-spends or incorrect totals. Comes with a performance cost.

---

## Autocommit

By default in PostgreSQL (and most databases), every statement outside an explicit `BEGIN...COMMIT` block is its own auto-committed transaction:

```sql
-- This is automatically wrapped in BEGIN/COMMIT:
UPDATE users SET last_login = NOW() WHERE user_id = 1042;
-- Committed immediately. No way to roll it back.
```

This is called **autocommit mode**. It's convenient but means individual statements have no safety net. For anything risky, always use an explicit `BEGIN`.

---

## Summary

```
+----------------------+-----------------------------------------------+
| Command              | Effect                                        |
+----------------------+-----------------------------------------------+
| BEGIN / START TRANS. | Open a transaction block                      |
| COMMIT               | Save all changes permanently                  |
| ROLLBACK             | Undo all changes since BEGIN                  |
| SAVEPOINT name       | Create a named checkpoint inside a transaction|
| ROLLBACK TO name     | Undo back to the named savepoint              |
| RELEASE SAVEPOINT    | Discard a savepoint (keep changes so far)     |
+----------------------+-----------------------------------------------+

ACID:
  Atomicity   = all or nothing
  Consistency = constraints always satisfied
  Isolation   = concurrent txns don't bleed into each other
  Durability  = commits survive crashes
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← UPDATE and DELETE](./update_and_delete.md) &nbsp;|&nbsp; **Next:** [Query Optimization →](../08_performance/query_optimization.md)

**Related Topics:** [INSERT](./insert.md) · [UPDATE and DELETE](./update_and_delete.md)

---

## 📝 Practice Questions

- 📝 [Q61 · acid-properties](../sql_practice_questions_100.md#q61--interview--acid-properties)
- 📝 [Q62 · transaction-basics](../sql_practice_questions_100.md#q62--normal--transaction-basics)
- 📝 [Q63 · savepoint](../sql_practice_questions_100.md#q63--normal--savepoint)
- 📝 [Q64 · isolation-levels](../sql_practice_questions_100.md#q64--interview--isolation-levels)
- 📝 [Q80 · explain-acid](../sql_practice_questions_100.md#q80--interview--explain-acid)
- 📝 [Q85 · blocking-transaction-scenario](../sql_practice_questions_100.md#q85--design--blocking-transaction-scenario)

