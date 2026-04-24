# Data Types

## Choosing the Right Container

Imagine a storage room full of containers — glass jars, paper bags, metal tins, wooden crates, refrigerated units. You would not store water in a paper bag. You would not put hot soup in a plastic carrier. You would not keep raw meat in a room-temperature ceramic pot and call it safe.

Database columns are the same. Every column is a container, and picking the wrong container causes bugs, performance problems, wasted storage, and silent data corruption. Storing a price as a floating-point number sounds fine until two monetary values that should add up to exactly £10.00 give you £9.999999999997. Storing a date as a string works until someone stores `"Jan 5th 2024"` in one row and `"2024-01-05"` in the next.

The database cannot enforce what it cannot understand. Choosing the right type is the first line of defence.

---

## Numeric Types

### Integers

Use integers for whole numbers — IDs, counts, quantities, ages.

```sql
user_id    SMALLINT   -- -32,768 to 32,767 (2 bytes)
stock_qty  INT        -- -2.1 billion to 2.1 billion (4 bytes) — the everyday default
page_views BIGINT     -- ~9.2 quintillion (8 bytes) — for very large counters
```

PostgreSQL also has `SERIAL`, `BIGSERIAL`, and `SMALLSERIAL` — auto-incrementing integer types that create a sequence automatically:

```sql
user_id SERIAL PRIMARY KEY   -- equivalent to INT + auto-increment sequence
```

> **MySQL note:** Use `INT AUTO_INCREMENT`. There is no `SERIAL` keyword.
> **SQLite note:** Use `INTEGER PRIMARY KEY` which is auto-incrementing.

### Exact Decimals (Money, Measurements)

```sql
price      DECIMAL(10, 2)   -- up to 10 digits total, 2 after decimal point
tax_rate   NUMERIC(5, 4)    -- e.g. 0.0875 for 8.75%
```

`DECIMAL` and `NUMERIC` are identical in PostgreSQL. They store exact values — no floating-point rounding errors.

**Critical rule: never use FLOAT or REAL for money.**

```sql
-- The floating-point trap
SELECT 0.1::FLOAT + 0.2::FLOAT;
-- Returns: 0.30000000000000004  (!)

SELECT 0.1::DECIMAL + 0.2::DECIMAL;
-- Returns: 0.3  (correct)
```

### Floating Point (Scientific Calculations)

```sql
latitude   FLOAT    -- IEEE 754 double precision (8 bytes)
weight_kg  REAL     -- IEEE 754 single precision (4 bytes, less precise)
```

Use these for scientific data, coordinates, or ML feature values where tiny imprecision is acceptable and range matters more than exactness.

---

## Text Types

### VARCHAR vs TEXT vs CHAR

```sql
username    VARCHAR(50)    -- variable length, max 50 characters
bio         TEXT           -- unlimited length
country_code CHAR(2)       -- fixed length, always 2 characters (e.g. 'GB', 'US')
```

```
+----------+----------------------------------------------------------+
| Type     | Use when...                                              |
+----------+----------------------------------------------------------+
| CHAR(n)  | Truly fixed-length strings (ISO codes, gender codes)     |
| VARCHAR(n)| Variable-length with a known maximum (names, emails)    |
| TEXT     | Unlimited-length content (descriptions, posts, logs)     |
+----------+----------------------------------------------------------+
```

> **Common mistake:** Using `TEXT` for everything. While PostgreSQL stores `TEXT` and `VARCHAR` identically under the hood, `VARCHAR(n)` documents intent and adds a constraint — a column declared `VARCHAR(255)` will reject any string longer than 255 characters, which catches bugs early.

> **MySQL note:** `TEXT` has subtypes: `TINYTEXT` (255 bytes), `TEXT` (64KB), `MEDIUMTEXT` (16MB), `LONGTEXT` (4GB). Also, MySQL `VARCHAR` has a 65,535 byte row limit.

---

## Date and Time Types

```sql
birth_date   DATE          -- year, month, day only: '2024-01-15'
start_time   TIME          -- time of day only: '14:30:00'
created_at   TIMESTAMP     -- date + time, no timezone: '2024-01-15 14:30:00'
updated_at   TIMESTAMPTZ   -- date + time WITH timezone: '2024-01-15 14:30:00+00'
```

### TIMESTAMP vs TIMESTAMPTZ

This is one of the most impactful decisions in schema design.

```
TIMESTAMP    stores the literal numbers you gave it — no timezone awareness.
             '2024-01-15 09:00:00' stored in London, read back in Tokyo = still 09:00.
             Misleading for global applications.

TIMESTAMPTZ  converts to UTC on storage, converts back to session timezone on retrieval.
             '2024-01-15 09:00:00+00' always means 9am UTC, no ambiguity.
```

**Recommendation: always use `TIMESTAMPTZ` for application timestamps.** Use `TIMESTAMP` only for things that are truly timezone-agnostic (scheduled tasks that should fire at the same clock time everywhere, for example).

```sql
-- Good practice
created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
updated_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()

-- Avoid for user-facing timestamps
created_at  TIMESTAMP    NOT NULL DEFAULT NOW()  -- no timezone = ambiguity
```

> **Common mistake: storing dates as strings.**

```sql
-- Bad: string dates cannot be sorted, compared, or arithmetically manipulated
order_date VARCHAR(20) DEFAULT '2024-01-15'

-- Good
order_date DATE NOT NULL DEFAULT CURRENT_DATE
```

---

## Boolean

```sql
is_active    BOOLEAN   NOT NULL DEFAULT TRUE
is_verified  BOOLEAN   NOT NULL DEFAULT FALSE
```

PostgreSQL accepts: `TRUE`/`FALSE`, `'true'`/`'false'`, `1`/`0`, `'t'`/`'f'`, `'yes'`/`'no'`.

> **MySQL note:** MySQL has no native BOOLEAN type — use `TINYINT(1)` (0 = false, 1 = true). MySQL 8 accepts `BOOL` as an alias.
> **SQLite note:** No native BOOLEAN — store as `INTEGER` (0/1).

---

## UUID

Universally Unique Identifiers are 128-bit values, typically formatted as `550e8400-e29b-41d4-a716-446655440000`. They are useful when you cannot use a sequential integer — for example, when generating IDs on the client before hitting the database, or when merging data from multiple systems.

```sql
-- Enable the uuid extension (PostgreSQL)
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

CREATE TABLE sessions (
    session_id  UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     INT           NOT NULL REFERENCES users(user_id),
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);
```

> **Trade-off:** UUIDs are larger (16 bytes vs 4 bytes for INT), and random UUIDs cause index fragmentation (every insert goes to a random position in the B-tree). Use UUID v7 or `ULID` if you want globally unique IDs that still sort chronologically.

---

## JSON and JSONB (PostgreSQL)

PostgreSQL can store and query JSON documents natively.

```sql
CREATE TABLE events (
    event_id    SERIAL        PRIMARY KEY,
    event_type  VARCHAR(50)   NOT NULL,
    payload     JSONB         NOT NULL,
    created_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Insert
INSERT INTO events (event_type, payload) VALUES (
    'user_signup',
    '{"user_id": 42, "plan": "pro", "source": "organic"}'
);

-- Query a JSONB field
SELECT payload->>'plan' AS plan
FROM events
WHERE payload->>'source' = 'organic';
```

### JSON vs JSONB

```
+--------+-----------------------------------------------------------+
| Type   | Characteristics                                           |
+--------+-----------------------------------------------------------+
| JSON   | Stored as plain text. Exact whitespace preserved.        |
|        | No indexing support. Slower queries.                     |
| JSONB  | Stored as binary (parsed). Whitespace normalised.        |
|        | Supports GIN indexes for fast key/value lookups.         |
|        | Slightly slower on INSERT, much faster on SELECT.        |
+--------+-----------------------------------------------------------+
```

**Recommendation: use `JSONB` in almost every case.** Use `JSON` only if you need to preserve exact whitespace or key ordering (very rare).

---

## Type Selection Quick Reference

```
+------------------------+---------------------------------+-----------------------------+
| What you are storing   | Recommended type                | Avoid                       |
+------------------------+---------------------------------+-----------------------------+
| Row identifier (auto)  | SERIAL / BIGSERIAL              | VARCHAR id, random string   |
| Distributed unique id  | UUID (+ gen_random_uuid())      | Manual random VARCHAR       |
| Whole number count     | INT                             | BIGINT (unless >2 billion)  |
| Money / price          | DECIMAL(10, 2)                  | FLOAT, REAL                 |
| Scientific measurement | FLOAT / REAL                    | DECIMAL (overkill)          |
| Short text (bounded)   | VARCHAR(n)                      | TEXT (when max is known)    |
| Long content           | TEXT                            | VARCHAR(5000) (arbitrary)   |
| Fixed code             | CHAR(2)                         | VARCHAR(2) (minor, ok)      |
| Yes/no flag            | BOOLEAN                         | INT, CHAR(1)                |
| Calendar date          | DATE                            | VARCHAR, INT (epoch days)   |
| Timestamp (global app) | TIMESTAMPTZ                     | TIMESTAMP, VARCHAR          |
| Flexible schema        | JSONB                           | Multiple nullable columns   |
+------------------------+---------------------------------+-----------------------------+
```

---

**[Back to README](../README.md)**

**Prev:** [← Tables and Constraints](./tables_and_constraints.md) &nbsp;|&nbsp; **Next:** [Normalization →](./normalization.md)

**Related Topics:** [Tables and Constraints](./tables_and_constraints.md) · [Normalization](./normalization.md) · [Indexes](./indexes.md)

---

## 📝 Practice Questions

- 📝 [Q33 · data-types](../sql_practice_questions_100.md#q33--interview--data-types)

