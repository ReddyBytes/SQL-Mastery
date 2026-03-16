# Stored Procedures — Reusable Logic in the Database

## The Recipe Card Analogy

A head chef doesn't recite the full pasta recipe from memory every single evening. They wrote it down on a laminated card months ago, filed it in the kitchen, and now any chef just calls "make_pasta" and follows the steps. Every cook produces the same result, every time.

A stored procedure (or function) is that recipe card — a named, reusable block of SQL logic stored inside the database. Instead of your application sending 10 separate SQL statements every time an order is placed, it calls one function: `place_order(user_id, cart_items)`. The logic lives in one place, executes close to the data, and can be called from any language or tool.

---

## PostgreSQL vs MySQL: A Key Difference

PostgreSQL uses `CREATE FUNCTION` with PL/pgSQL (its procedural language). MySQL has both `CREATE PROCEDURE` and `CREATE FUNCTION` with distinct syntax. This chapter covers both.

```
+--------------------+----------------------------+---------------------------+
| Feature            | PostgreSQL                 | MySQL                     |
+--------------------+----------------------------+---------------------------+
| Procedure syntax   | CREATE FUNCTION            | CREATE PROCEDURE          |
| Called with        | SELECT func() or CALL func | CALL procedure_name()     |
| Language           | PL/pgSQL, SQL, Python, etc | SQL + procedural SQL      |
| Return values      | RETURNS <type>             | OUT parameters / result   |
| Error handling     | RAISE / EXCEPTION          | SIGNAL / RESIGNAL         |
+--------------------+----------------------------+---------------------------+
```

---

## PostgreSQL: CREATE FUNCTION (PL/pgSQL)

### Basic Function — No Return Value (RETURNS void)

```sql
CREATE OR REPLACE FUNCTION deactivate_user(p_user_id INT)
RETURNS void
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE users
    SET    is_active  = FALSE,
           deleted_at = NOW()
    WHERE  user_id = p_user_id;

    -- Raise an informational notice (visible to the caller)
    RAISE NOTICE 'User % has been deactivated', p_user_id;
END;
$$;

-- Call it
SELECT deactivate_user(1042);
```

### Function That Returns a Value

```sql
CREATE OR REPLACE FUNCTION get_customer_lifetime_value(p_user_id INT)
RETURNS NUMERIC(10,2)
LANGUAGE plpgsql
AS $$
DECLARE
    v_total NUMERIC(10,2);
BEGIN
    SELECT COALESCE(SUM(total_amount), 0)
    INTO   v_total
    FROM   orders
    WHERE  user_id = p_user_id
      AND  status  = 'completed';

    RETURN v_total;
END;
$$;

-- Use it like any expression
SELECT user_id, name, get_customer_lifetime_value(user_id) AS ltv
FROM   users
WHERE  is_active = TRUE
ORDER  BY ltv DESC
LIMIT  10;
```

### Function That Returns a Table

```sql
CREATE OR REPLACE FUNCTION get_user_orders(p_user_id INT)
RETURNS TABLE (
    order_id     INT,
    status       TEXT,
    total_amount NUMERIC(10,2),
    created_at   TIMESTAMPTZ
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT o.order_id, o.status, o.total_amount, o.created_at
    FROM   orders o
    WHERE  o.user_id = p_user_id
    ORDER  BY o.created_at DESC;
END;
$$;

-- Query it like a table
SELECT * FROM get_user_orders(1042);
```

---

## Error Handling with RAISE and EXCEPTION

```sql
CREATE OR REPLACE FUNCTION transfer_funds(
    p_from_account INT,
    p_to_account   INT,
    p_amount       NUMERIC(10,2)
)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
    v_balance NUMERIC(10,2);
BEGIN
    -- Validate input
    IF p_amount <= 0 THEN
        RAISE EXCEPTION 'Transfer amount must be positive, got: %', p_amount;
    END IF;

    -- Check balance
    SELECT balance INTO v_balance
    FROM   accounts
    WHERE  account_id = p_from_account
    FOR    UPDATE;  -- lock the row during the transaction

    IF v_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: balance is %, requested %',
            v_balance, p_amount;
    END IF;

    -- Perform transfer
    UPDATE accounts SET balance = balance - p_amount WHERE account_id = p_from_account;
    UPDATE accounts SET balance = balance + p_amount WHERE account_id = p_to_account;

    RAISE NOTICE 'Transferred % from account % to account %',
        p_amount, p_from_account, p_to_account;
END;
$$;
```

---

## Real-World: Place an Order Function

A single function that handles the full order placement — inserts the order, line items, and deducts inventory atomically:

```sql
CREATE OR REPLACE FUNCTION place_order(
    p_user_id    INT,
    p_product_id INT,
    p_quantity   INT
)
RETURNS INT   -- returns the new order_id
LANGUAGE plpgsql
AS $$
DECLARE
    v_price      NUMERIC(10,2);
    v_stock      INT;
    v_order_id   INT;
    v_total      NUMERIC(10,2);
BEGIN
    -- Get product price and stock, lock the row
    SELECT price, stock_qty
    INTO   v_price, v_stock
    FROM   products
    WHERE  product_id = p_product_id
    FOR    UPDATE;

    IF v_stock < p_quantity THEN
        RAISE EXCEPTION 'Not enough stock for product %: % available, % requested',
            p_product_id, v_stock, p_quantity;
    END IF;

    v_total := v_price * p_quantity;

    -- Insert the order
    INSERT INTO orders (user_id, status, total_amount, created_at)
    VALUES (p_user_id, 'pending', v_total, NOW())
    RETURNING order_id INTO v_order_id;

    -- Insert the order item
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (v_order_id, p_product_id, p_quantity, v_price);

    -- Deduct inventory
    UPDATE products
    SET    stock_qty = stock_qty - p_quantity
    WHERE  product_id = p_product_id;

    RETURN v_order_id;
END;
$$;

-- Place an order for user 1042: 3 units of product 88
SELECT place_order(1042, 88, 3);
```

---

## MySQL: CREATE PROCEDURE Syntax

For completeness, here's the MySQL equivalent using its `PROCEDURE` syntax:

```sql
DELIMITER $$

CREATE PROCEDURE deactivate_user(IN p_user_id INT)
BEGIN
    UPDATE users
    SET    is_active  = 0,
           deleted_at = NOW()
    WHERE  user_id = p_user_id;

    SELECT ROW_COUNT() AS rows_affected;
END$$

DELIMITER ;

-- Call it
CALL deactivate_user(1042);
```

```sql
DELIMITER $$

CREATE PROCEDURE get_customer_ltv(IN p_user_id INT, OUT p_total DECIMAL(10,2))
BEGIN
    SELECT COALESCE(SUM(total_amount), 0)
    INTO   p_total
    FROM   orders
    WHERE  user_id  = p_user_id
      AND  status   = 'completed';
END$$

DELIMITER ;

-- Call it
CALL get_customer_ltv(1042, @ltv);
SELECT @ltv;
```

---

## CALL syntax (PostgreSQL 11+)

PostgreSQL 11 added `CREATE PROCEDURE` + `CALL` syntax alongside the existing function approach:

```sql
-- PostgreSQL 11+: CREATE PROCEDURE (for procedures that don't return values)
CREATE OR REPLACE PROCEDURE archive_old_orders(p_cutoff_date DATE)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO orders_archive
    SELECT * FROM orders WHERE created_at < p_cutoff_date;

    DELETE FROM orders WHERE created_at < p_cutoff_date;

    COMMIT;  -- procedures (not functions) can COMMIT inside their body
END;
$$;

CALL archive_old_orders('2022-01-01');
```

---

## Dropping Functions and Procedures

```sql
-- Drop a function (must specify argument types due to overloading)
DROP FUNCTION IF EXISTS get_customer_lifetime_value(INT);

-- Drop a procedure
DROP PROCEDURE IF EXISTS deactivate_user(INT);
```

---

## Summary

```
+--------------------------------+----------------------------------------------+
| Syntax                         | Effect                                       |
+--------------------------------+----------------------------------------------+
| CREATE OR REPLACE FUNCTION     | Create/update a PL/pgSQL function            |
| RETURNS void                   | Function that performs work, returns nothing |
| RETURNS <type>                 | Function that returns a scalar value         |
| RETURNS TABLE (...)            | Function that returns a result set           |
| RETURN QUERY SELECT ...        | Return results of a SELECT from a function  |
| RAISE NOTICE / EXCEPTION       | Log a message / raise an error              |
| CREATE PROCEDURE (PG 11+)      | Procedure that can COMMIT internally         |
| CALL procedure_name(args)      | Execute a procedure                          |
+--------------------------------+----------------------------------------------+

PostgreSQL:  CREATE FUNCTION + SELECT func() or CALL func()
MySQL:       CREATE PROCEDURE + CALL, or CREATE FUNCTION + SELECT
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Views](./views.md) &nbsp;|&nbsp; **Next:** [Triggers →](./triggers.md)

**Related Topics:** [Views](./views.md) · [Triggers](./triggers.md) · [Transactions](../07_data_modification/transactions.md)
