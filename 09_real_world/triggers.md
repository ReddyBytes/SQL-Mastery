# Triggers — Automatic Actions When Data Changes

## The Motion Sensor Analogy

A motion sensor light in a car park doesn't require you to flip a switch. The moment you walk in, it turns on automatically. You didn't ask for it. You didn't even think about it. The sensor detected an event (motion) and reacted.

A database trigger works the same way. When a row is inserted, updated, or deleted in a table, the trigger fires automatically — running a pre-defined function before or after the event. Your application code never has to remember to do it.

This is powerful. It's also dangerous. An invisible side effect that fires silently on every update is exactly the kind of thing that causes debugging nightmares when someone forgets it's there. Use triggers purposefully, and document them well.

---

## How Triggers Work

```
Event (INSERT / UPDATE / DELETE)
          |
          v
    BEFORE trigger (optional)   <-- can modify the row before it's saved
          |
          v
    The actual data change
          |
          v
    AFTER trigger (optional)    <-- reacts to the confirmed change
```

PostgreSQL triggers require two parts:
1. A **trigger function** (returns `TRIGGER`) that contains the logic
2. A **trigger definition** that binds the function to a table and event

---

## BEFORE vs AFTER Triggers

```
+----------------+-----------------------------------------------------------+
| BEFORE         | Fires before the change is written                        |
|                | Can modify the row (by returning NEW with changes)         |
|                | Can cancel the operation (by returning NULL)               |
|                | Use for: validation, auto-setting fields                   |
+----------------+-----------------------------------------------------------+
| AFTER          | Fires after the change is committed to the table          |
|                | Cannot modify the triggering row                          |
|                | Use for: audit logs, cascading updates to other tables    |
+----------------+-----------------------------------------------------------+
```

---

## FOR EACH ROW vs FOR EACH STATEMENT

```sql
FOR EACH ROW       -- fires once per affected row (most common)
FOR EACH STATEMENT -- fires once per SQL statement, regardless of row count
```

`FOR EACH STATEMENT` is rarely used but useful for logging that a batch operation occurred.

---

## Real-World Example 1: Auto-Update `updated_at`

This is the most common trigger in production databases. Rather than requiring every UPDATE query to manually set `updated_at = NOW()`, a BEFORE trigger handles it automatically.

```sql
-- Step 1: Create the trigger function
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := NOW();
    RETURN NEW;   -- must return the (modified) row for BEFORE triggers
END;
$$;

-- Step 2: Attach it to the products table
CREATE TRIGGER trg_products_updated_at
BEFORE UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

-- Step 3: Repeat for any table that has updated_at
CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

-- Now: every UPDATE on products automatically sets updated_at
UPDATE products SET price = 34.99 WHERE product_id = 88;
-- updated_at was set to NOW() without you writing it
```

---

## Real-World Example 2: Price Change Audit Log

When a product's price changes, write a record to an audit table — who changed it, what it was, what it became.

```sql
-- Step 1: Create the audit table
CREATE TABLE product_price_audit (
    audit_id    SERIAL PRIMARY KEY,
    product_id  INT          NOT NULL,
    old_price   NUMERIC(10,2),
    new_price   NUMERIC(10,2),
    changed_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    changed_by  TEXT         NOT NULL DEFAULT current_user
);

-- Step 2: Create the trigger function
CREATE OR REPLACE FUNCTION log_price_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Only fire if the price actually changed
    IF OLD.price IS DISTINCT FROM NEW.price THEN
        INSERT INTO product_price_audit
            (product_id, old_price, new_price, changed_at, changed_by)
        VALUES
            (NEW.product_id, OLD.price, NEW.price, NOW(), current_user);
    END IF;

    RETURN NEW;
END;
$$;

-- Step 3: Create the trigger
CREATE TRIGGER trg_product_price_audit
AFTER UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION log_price_change();
```

Now every price update is permanently recorded:

```sql
-- Change a price
UPDATE products SET price = 39.99 WHERE product_id = 88;

-- Check the audit trail
SELECT * FROM product_price_audit WHERE product_id = 88;
```

```
 audit_id | product_id | old_price | new_price |         changed_at         | changed_by
----------+------------+-----------+-----------+----------------------------+------------
        1 |         88 |     29.99 |     39.99 | 2024-06-15 14:32:01.234+00 | app_user
        2 |         88 |     39.99 |     34.99 | 2024-08-20 09:11:55.891+00 | admin
```

---

## BEFORE INSERT Trigger: Validation

Use a BEFORE INSERT trigger to enforce business rules that are too complex for a simple CHECK constraint.

```sql
CREATE OR REPLACE FUNCTION validate_order_amount()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF NEW.total_amount <= 0 THEN
        RAISE EXCEPTION 'Order total must be positive: got %', NEW.total_amount;
    END IF;

    IF NEW.total_amount > 100000 THEN
        RAISE EXCEPTION 'Order total exceeds maximum allowed (100,000): got %',
            NEW.total_amount;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_orders_validate_amount
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION validate_order_amount();
```

---

## The `NEW` and `OLD` Variables

Inside a trigger function, two special record variables are available:

```
+----------+---------------+--------------+--------------------+
| Variable | INSERT        | UPDATE       | DELETE             |
+----------+---------------+--------------+--------------------+
| NEW      | New row data  | New row data | NULL (no new row)  |
| OLD      | NULL          | Old row data | Old row data       |
+----------+---------------+--------------+--------------------+
```

```sql
-- In a DELETE trigger, OLD holds the row being deleted
CREATE OR REPLACE FUNCTION log_deleted_user()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO deleted_users_log (user_id, email, deleted_at)
    VALUES (OLD.user_id, OLD.email, NOW());
    RETURN OLD;
END;
$$;

CREATE TRIGGER trg_users_delete_log
AFTER DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION log_deleted_user();
```

---

## Managing Triggers

```sql
-- List all triggers on a table
SELECT trigger_name, event_manipulation, action_timing
FROM   information_schema.triggers
WHERE  event_object_table = 'products';

-- Disable a trigger temporarily (useful during bulk data loads)
ALTER TABLE products DISABLE TRIGGER trg_product_price_audit;

-- Re-enable it
ALTER TABLE products ENABLE TRIGGER trg_product_price_audit;

-- Disable ALL triggers on a table (requires superuser in PostgreSQL)
ALTER TABLE products DISABLE TRIGGER ALL;

-- Drop a trigger
DROP TRIGGER IF EXISTS trg_product_price_audit ON products;
-- Note: the trigger function remains; drop it separately if needed
DROP FUNCTION IF EXISTS log_price_change();
```

---

## Warnings: Use Triggers Sparingly

Triggers are powerful, but they come with real risks:

```
+----------------------------+------------------------------------------------+
| Risk                       | Mitigation                                     |
+----------------------------+------------------------------------------------+
| Invisible side effects     | Document every trigger in code comments        |
|                            | and in a team wiki                             |
+----------------------------+------------------------------------------------+
| Cascading triggers         | Trigger A fires → updates table B → fires      |
| (triggers calling triggers)| trigger B → updates table C → ... nightmare    |
+----------------------------+------------------------------------------------+
| Performance overhead       | Every INSERT/UPDATE fires the function —       |
|                            | keep trigger logic simple and fast             |
+----------------------------+------------------------------------------------+
| Hard to debug              | Errors appear to come from the original query; |
|                            | always check for trigger-related errors first  |
+----------------------------+------------------------------------------------+
| Skipped during bulk loads  | COPY and TRUNCATE may not fire row-level       |
|                            | triggers — verify for your data pipeline       |
+----------------------------+------------------------------------------------+
```

---

## Summary

```
+-----------------------------------+------------------------------------------+
| Concept                           | Details                                  |
+-----------------------------------+------------------------------------------+
| CREATE OR REPLACE FUNCTION        | Define the trigger's logic (returns TRIGGER)|
| CREATE TRIGGER name BEFORE/AFTER  | Bind the function to a table + event     |
| FOR EACH ROW                      | Fires once per affected row              |
| FOR EACH STATEMENT                | Fires once per SQL statement             |
| NEW                               | The row being inserted or updated        |
| OLD                               | The row before update, or the deleted row|
| RETURN NEW                        | Required in BEFORE triggers to save row  |
| RETURN NULL                       | In BEFORE trigger: cancels the operation |
| DISABLE TRIGGER name ON table     | Temporarily disable a trigger            |
+-----------------------------------+------------------------------------------+

Best use cases: auto-set timestamps, audit logs, input validation
Avoid for: complex business logic, anything better handled in app code
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Stored Procedures](./stored_procedures.md) &nbsp;|&nbsp; **Next:** [SQL with Python →](./sql_with_python.md)

**Related Topics:** [Views](./views.md) · [Stored Procedures](./stored_procedures.md) · [INSERT](../07_data_modification/insert.md)
