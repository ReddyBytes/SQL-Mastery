# String and Date Functions — Power Tools for Text and Time

## The Analogy

A woodworker has a toolbox full of power tools — a jigsaw for cutting shapes, a router for clean edges, a sander for finishing. Each tool has a specific job; you wouldn't use a sander to cut a board.

String and date functions are your SQL power tools. `TRIM` strips unwanted whitespace (sander). `SUBSTRING` cuts out the part you want (jigsaw). `DATE_TRUNC` rounds a timestamp down to the nearest month (router for time). You reach for the right tool for the job rather than writing complex procedural logic in application code.

---

## Setup

```sql
CREATE TABLE customers (
    id         SERIAL PRIMARY KEY,
    full_name  TEXT,
    email      TEXT,
    city       TEXT,
    signed_up  TIMESTAMPTZ
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT,
    product     TEXT,
    total       NUMERIC(10,2),
    created_at  TIMESTAMPTZ
);

INSERT INTO customers VALUES
    (1, '  alice johnson ', 'Alice.Johnson@Example.COM', 'London',  '2023-01-15 09:30:00+00'),
    (2, 'Bob Smith',        'bob.smith@example.com',     'Paris',   '2023-03-22 14:00:00+00'),
    (3, 'Carol   White',    'carol.white@example.com',   'Berlin',  '2023-06-01 11:15:00+00'),
    (4, 'Dave Brown',       'dave.brown@example.com',    'Madrid',  '2024-01-10 08:45:00+00');

INSERT INTO orders VALUES
    (1, 1,  'Laptop',     1299.00, '2024-01-05 10:00:00+00'),
    (2, 1,  'Mouse',        25.99, '2024-02-10 16:30:00+00'),
    (3, 2,  'Keyboard',     89.00, '2024-02-20 09:00:00+00'),
    (4, 3,  'Monitor',     349.00, '2024-03-01 13:00:00+00'),
    (5, 3,  'Webcam',       59.99, '2024-03-15 17:45:00+00'),
    (6, 4,  'Headphones',  120.00, '2024-04-01 12:00:00+00');
```

---

## String Functions

### LENGTH / CHAR_LENGTH

```sql
SELECT
    full_name,
    LENGTH(full_name)      AS len_with_spaces,
    LENGTH(TRIM(full_name)) AS len_trimmed
FROM customers;
```

> **MySQL note:** `CHAR_LENGTH` counts characters (correct for multibyte); `LENGTH` counts bytes. In PostgreSQL, `LENGTH` counts characters for TEXT columns.

---

### UPPER / LOWER

Normalise case before comparing or displaying:

```sql
-- Normalise email to lowercase for deduplication
SELECT LOWER(email) AS normalised_email FROM customers;

-- Case-insensitive search
SELECT * FROM customers
WHERE LOWER(email) = LOWER('Alice.Johnson@Example.COM');
```

---

### TRIM / LTRIM / RTRIM

```sql
-- Fix messy imported data: '  alice johnson ' → 'alice johnson'
SELECT
    full_name,
    TRIM(full_name)         AS trimmed,
    LTRIM(full_name)        AS left_trimmed,
    RTRIM(full_name)        AS right_trimmed,
    TRIM(BOTH 'x' FROM 'xxxhelloxxx') AS custom_trim  -- trims specific char
FROM customers;
```

---

### SUBSTRING / SUBSTR

```sql
-- Extract domain from email address
SELECT
    email,
    SUBSTRING(email FROM POSITION('@' IN email) + 1) AS domain
FROM customers;

-- First 3 chars of city
SELECT city, SUBSTRING(city FROM 1 FOR 3) AS city_code FROM customers;
```

> **MySQL note:** MySQL uses `SUBSTR(str, start, length)` — same function, different syntax. Both work in PostgreSQL too: `SUBSTR('London', 1, 3)`.

---

### REPLACE

```sql
-- Mask middle of email for display
SELECT REPLACE(email, SUBSTRING(email FROM 2 FOR 3), '***') AS masked_email
FROM customers;

-- Clean up double spaces in imported text
SELECT REPLACE(full_name, '   ', ' ') AS cleaned_name FROM customers;
```

---

### CONCAT / ||

```sql
-- Build a display name from parts
SELECT
    CONCAT(TRIM(full_name), ' <', LOWER(email), '>') AS display,
    -- PostgreSQL also supports the || operator:
    TRIM(full_name) || ' (' || city || ')' AS name_with_city
FROM customers;
```

> **MySQL note:** MySQL's `||` is logical OR by default (use `CONCAT` instead, or set `PIPES_AS_CONCAT` mode). PostgreSQL's `||` is string concatenation.

---

### LIKE and ILIKE

```sql
-- LIKE is case-sensitive
SELECT * FROM customers WHERE full_name LIKE '%johnson%';   -- no match (capital J)
SELECT * FROM customers WHERE full_name LIKE '%Johnson%';   -- matches Alice

-- ILIKE is PostgreSQL's case-insensitive LIKE
SELECT * FROM customers WHERE full_name ILIKE '%johnson%';  -- matches Alice

-- Pattern wildcards:
--   %  = zero or more characters
--   _  = exactly one character
SELECT * FROM customers WHERE email LIKE '%.com';           -- all .com emails
SELECT * FROM customers WHERE city  LIKE 'L____n';          -- L + 4 chars + n = London
```

> **MySQL note:** MySQL's `LIKE` is case-insensitive by default for `utf8mb4_general_ci` collation. There is no `ILIKE`.

---

### REGEXP / SIMILAR TO

```sql
-- PostgreSQL REGEXP (~ operator)
SELECT * FROM customers WHERE email ~ '^[a-z]+\.[a-z]+@';   -- first.last format
SELECT * FROM customers WHERE email ~* '^alice';             -- ~* is case-insensitive

-- SIMILAR TO (SQL standard, less powerful than full regex)
SELECT * FROM customers WHERE city SIMILAR TO '(London|Paris|Berlin)';
```

---

### POSITION / STRPOS

```sql
SELECT
    email,
    POSITION('@' IN email)   AS at_position,
    STRPOS(email, '@')       AS at_position_alt   -- same result
FROM customers;
```

---

## Date and Time Functions

### NOW(), CURRENT_DATE, CURRENT_TIMESTAMP

```sql
SELECT
    NOW()              AS now_with_tz,           -- current timestamp with timezone
    CURRENT_TIMESTAMP  AS current_ts,            -- same as NOW()
    CURRENT_DATE       AS today,                 -- date only, no time
    CURRENT_TIME       AS time_now;              -- time only
```

---

### DATE_TRUNC — Round Down to a Precision

The most-used date function in analytics:

```sql
-- Group orders by month for monthly revenue report
SELECT
    DATE_TRUNC('month', created_at) AS month,
    COUNT(*)                        AS order_count,
    SUM(total)                      AS revenue
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;
```

```
         month          | order_count | revenue
------------------------+-------------+---------
 2024-01-01 00:00:00+00 |           1 | 1299.00
 2024-02-01 00:00:00+00 |           2 |  114.99
 2024-03-01 00:00:00+00 |           2 |  408.99
 2024-04-01 00:00:00+00 |           1 |  120.00
```

Precision values: `'microsecond'`, `'millisecond'`, `'second'`, `'minute'`, `'hour'`, `'day'`, `'week'`, `'month'`, `'quarter'`, `'year'`.

> **MySQL note:** MySQL uses `DATE_FORMAT(created_at, '%Y-%m')` to group by month. PostgreSQL uses `DATE_TRUNC` or `TO_CHAR`.

---

### EXTRACT — Pull Out One Part of a Date

```sql
SELECT
    created_at,
    EXTRACT(YEAR   FROM created_at) AS yr,
    EXTRACT(MONTH  FROM created_at) AS mo,
    EXTRACT(DOW    FROM created_at) AS day_of_week,  -- 0=Sunday, 6=Saturday
    EXTRACT(HOUR   FROM created_at) AS hr,
    EXTRACT(EPOCH  FROM created_at) AS unix_seconds
FROM orders;
```

> **MySQL note:** MySQL uses `YEAR(date)`, `MONTH(date)`, `DAYOFWEEK(date)` as separate functions.

---

### AGE — Human-Readable Interval Between Dates

```sql
-- How long has each customer been signed up?
SELECT
    name,
    signed_up,
    AGE(NOW(), signed_up::TIMESTAMPTZ) AS tenure
FROM customers;
```

```
 name  | signed_up  | tenure
-------+------------+-------------------
 Alice | 2023-01-15 | 2 years 2 mons ...
 Bob   | 2023-03-22 | 1 year 11 mons ...
```

---

### Date Arithmetic and INTERVAL

```sql
-- Orders placed in the last 30 days
SELECT * FROM orders
WHERE created_at >= NOW() - INTERVAL '30 days';

-- Add/subtract fixed periods
SELECT
    created_at,
    created_at + INTERVAL '1 year'  AS one_year_later,
    created_at - INTERVAL '7 days'  AS one_week_before
FROM orders;

-- Difference in days between two dates (PostgreSQL)
SELECT
    id,
    (NOW()::DATE - created_at::DATE) AS days_since_order
FROM orders;
```

> **MySQL note:** MySQL has `DATEDIFF(date1, date2)` for day differences and `DATE_ADD(date, INTERVAL n unit)` for arithmetic. PostgreSQL uses plain subtraction and INTERVAL literals.

---

### TO_CHAR — Format a Date as a String

```sql
-- Human-friendly date labels for reports
SELECT
    created_at,
    TO_CHAR(created_at, 'DD Mon YYYY')       AS formatted_date,   -- 05 Jan 2024
    TO_CHAR(created_at, 'YYYY-MM')           AS year_month,       -- 2024-01
    TO_CHAR(created_at, 'Day, DD Month YYYY') AS long_format      -- Tuesday, 05 January 2024
FROM orders;
```

> **MySQL note:** MySQL's equivalent is `DATE_FORMAT(date, '%d %b %Y')`.

---

### Timezone Handling

```sql
-- Convert a UTC timestamp to a local timezone
SELECT
    created_at                                      AS utc,
    created_at AT TIME ZONE 'Europe/London'         AS london_time,
    created_at AT TIME ZONE 'America/New_York'      AS ny_time
FROM orders;

-- Store timezone-aware vs timezone-naive
-- TIMESTAMPTZ stores UTC internally, displays in session timezone
-- TIMESTAMP    has no timezone info — risky for global apps
```

---

## Real-World: Monthly Revenue Report with Parsed Dates

```sql
-- Monthly revenue summary with human-readable month labels
SELECT
    TO_CHAR(DATE_TRUNC('month', created_at), 'Mon YYYY') AS month_label,
    COUNT(DISTINCT customer_id)                          AS unique_buyers,
    COUNT(*)                                             AS order_count,
    SUM(total)                                           AS revenue,
    ROUND(AVG(total), 2)                                 AS avg_order_value
FROM orders
WHERE created_at >= DATE_TRUNC('year', NOW())   -- year to date
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY DATE_TRUNC('month', created_at);
```

---

## PostgreSQL vs MySQL Quick Reference

```
┌─────────────────────────┬──────────────────────────┬────────────────────────┐
│ Task                    │ PostgreSQL                │ MySQL                  │
├─────────────────────────┼──────────────────────────┼────────────────────────┤
│ String concat           │ || or CONCAT()            │ CONCAT() (not ||)      │
│ Case-insensitive LIKE   │ ILIKE                     │ LIKE (default ci)      │
│ Regex match             │ ~ and ~*                  │ REGEXP or RLIKE        │
│ Truncate to month       │ DATE_TRUNC('month', d)    │ DATE_FORMAT(d,'%Y-%m') │
│ Format date as text     │ TO_CHAR(d, 'YYYY-MM-DD')  │ DATE_FORMAT(d,'%Y-%m-%d')│
│ Day difference          │ date1 - date2             │ DATEDIFF(d1, d2)       │
│ Add time period         │ d + INTERVAL '1 month'    │ DATE_ADD(d,INTERVAL 1 MONTH)│
│ Extract year            │ EXTRACT(YEAR FROM d)      │ YEAR(d)                │
│ Current timestamp       │ NOW() or CURRENT_TIMESTAMP│ NOW()                  │
│ Timezone conversion     │ AT TIME ZONE 'tz'         │ CONVERT_TZ(d, from, to)│
└─────────────────────────┴──────────────────────────┴────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│              STRING AND DATE FUNCTIONS CHEAT SHEET                  │
├──────────────────────────┬──────────────────────────────────────────┤
│ LENGTH(s)                │ Character count                          │
│ UPPER/LOWER(s)           │ Change case                              │
│ TRIM / LTRIM / RTRIM     │ Remove whitespace (or chars) from edges  │
│ SUBSTRING(s FROM n FOR l)│ Extract part of a string                │
│ REPLACE(s, from, to)     │ Substitute text                          │
│ CONCAT / ||              │ Join strings together                    │
│ LIKE / ILIKE             │ Pattern matching (ILIKE = case-insensitive│
│ POSITION('@' IN s)       │ Find character position                  │
│ DATE_TRUNC('month', d)   │ Round timestamp down to precision        │
│ EXTRACT(YEAR FROM d)     │ Pull one field out of a date             │
│ AGE(d1, d2)              │ Human-readable interval                  │
│ d +/- INTERVAL '30 days' │ Date arithmetic                          │
│ TO_CHAR(d, 'format')     │ Date → formatted string                  │
│ AT TIME ZONE 'tz'        │ Convert timestamp to timezone            │
└──────────────────────────┴──────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← CASE and Conditionals](./case_and_conditionals.md) &nbsp;|&nbsp; **Next:** [INSERT →](../07_data_modification/insert.md)

**Related Topics:** [CASE and Conditionals](./case_and_conditionals.md) · [Subqueries](./subqueries.md) · [CTEs](./ctes.md)

---

## 📝 Practice Questions

- 📝 [Q54 · string-functions](../sql_practice_questions_100.md#q54--normal--string-functions)
- 📝 [Q55 · date-functions](../sql_practice_questions_100.md#q55--normal--date-functions)

