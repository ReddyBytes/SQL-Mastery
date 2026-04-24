# WHERE and Filtering

## The Amazon Search Filter Analogy

You open Amazon and search "headphones." Ten thousand results. Nobody wants to scroll through
all of them. So you click filters: **Under $100** · **4 stars and up** · **Wireless** ·
**In Stock**.

Instantly: 42 results. Exactly what you wanted.

That's the `WHERE` clause. Your table has ten thousand rows, but you only care about the ones
that match specific conditions. `WHERE` is the filter panel for your data.

```sql
-- 10,000 products → "wireless headphones under $100, 4+ stars, in stock"
SELECT product_name, price, rating
FROM   products
WHERE  category        = 'headphones'
  AND  wireless        = true
  AND  price           < 100
  AND  rating          >= 4.0
  AND  stock_quantity  > 0;
```

---

## The Products Table We'll Use

```
  TABLE: products
  ┌────┬──────────────────────────┬──────────┬────────┬──────────────┬──────────┐
  │ id │ product_name             │ category │ price  │ stock_qty    │ rating   │
  ├────┼──────────────────────────┼──────────┼────────┼──────────────┼──────────┤
  │  1 │ Sony WH-1000XM5          │ audio    │  279.99│     45       │  4.8     │
  │  2 │ JBL Clip 4               │ audio    │   59.99│    120       │  4.5     │
  │  3 │ Logitech MX Keys         │ keyboard │  109.99│     30       │  4.7     │
  │  4 │ Mechanical KB Pro        │ keyboard │   79.99│      0       │  4.2     │
  │  5 │ USB-C Hub 7-in-1         │ hub      │   49.99│     80       │  3.9     │
  │  6 │ Dell 27" Monitor         │ monitor  │  349.99│     12       │  4.6     │
  │  7 │ Generic HDMI Cable       │ cable    │    9.99│   NULL       │  NULL    │
  └────┴──────────────────────────┴──────────┴────────┴──────────────┴──────────┘
```

---

## Comparison Operators

The basic building blocks of every `WHERE` clause:

```
  Operator   Meaning                    Example
  ──────────────────────────────────────────────────────────────────
  =          Equal to                   WHERE category = 'audio'
  != or <>   Not equal to               WHERE category != 'cable'
  >          Greater than               WHERE price > 100
  <          Less than                  WHERE price < 50
  >=         Greater than or equal      WHERE rating >= 4.5
  <=         Less than or equal         WHERE stock_qty <= 10
```

```sql
-- Products that cost more than $100
SELECT product_name, price
FROM   products
WHERE  price > 100;
```

Result: Sony WH-1000XM5 (279.99), Logitech MX Keys (109.99), Dell 27" Monitor (349.99).

---

## BETWEEN — Range Filtering

`BETWEEN low AND high` is inclusive on both ends. It's shorthand for `>= low AND <= high`:

```sql
-- Products priced between $50 and $200 (inclusive)
SELECT product_name, price
FROM   products
WHERE  price BETWEEN 50 AND 200;
```

Result: Sony WH-1000XM5 would be excluded (279.99 > 200). JBL Clip 4 is in ($59.99).
Logitech MX Keys is in ($109.99).

Works on dates too:

```sql
-- Orders placed in Q1 2024
SELECT id, customer_id, total_amount
FROM   orders
WHERE  created_at BETWEEN '2024-01-01' AND '2024-03-31';
```

---

## IN — Match Any Value in a List

Instead of chaining multiple `OR` conditions, use `IN`:

```sql
-- Without IN (verbose):
WHERE category = 'audio' OR category = 'keyboard' OR category = 'monitor'

-- With IN (clean):
WHERE category IN ('audio', 'keyboard', 'monitor')
```

```sql
SELECT product_name, category
FROM   products
WHERE  category IN ('audio', 'keyboard');
```

Result: Sony WH-1000XM5, JBL Clip 4, Logitech MX Keys, Mechanical KB Pro.

`NOT IN` is the opposite:

```sql
-- Everything except audio and cable
SELECT product_name, category
FROM   products
WHERE  category NOT IN ('audio', 'cable');
```

> **Caution:** `NOT IN` behaves unexpectedly when the list contains a NULL value — it returns
> zero rows. This is a common bug. If your subquery could return NULLs, use `NOT EXISTS` instead.

---

## LIKE — Pattern Matching

`LIKE` matches text patterns using two wildcard characters:
- `%` matches any sequence of zero or more characters
- `_` matches exactly one character

```sql
-- Product names starting with "Sony"
SELECT product_name
FROM   products
WHERE  product_name LIKE 'Sony%';

-- Product names containing "Hub"
SELECT product_name
FROM   products
WHERE  product_name LIKE '%Hub%';

-- 4-character categories
SELECT product_name, category
FROM   products
WHERE  category LIKE '____';    -- 4 underscores → 4 chars (e.g. "hub!")
```

`ILIKE` in PostgreSQL is case-insensitive LIKE:

```sql
-- Matches "Sony", "sony", "SONY"
WHERE product_name ILIKE 'sony%'
```

> **MySQL note:** MySQL's `LIKE` is case-insensitive by default depending on collation.
> Use `LIKE BINARY 'Sony%'` for case-sensitive matching.

---

## IS NULL and IS NOT NULL

NULL means "value unknown" — not zero, not an empty string. It's the absence of a value.

The `Generic HDMI Cable` row has `stock_qty = NULL` and `rating = NULL`. To find these rows:

```sql
-- Products where stock quantity is unknown
SELECT product_name
FROM   products
WHERE  stock_qty IS NULL;
```

To find rows where a value *does* exist:

```sql
-- Products with a known rating
SELECT product_name, rating
FROM   products
WHERE  rating IS NOT NULL;
```

**Common mistake — never do this:**

```sql
-- WRONG: NULL = NULL is never true in SQL
WHERE stock_qty = NULL     -- Returns zero rows, always!

-- CORRECT:
WHERE stock_qty IS NULL
```

This trips up nearly every SQL beginner. Burn it into memory: **use `IS NULL`, never `= NULL`.**

---

## AND, OR, NOT — Combining Conditions

You can stack conditions using logical operators:

```sql
-- Audio products that are in stock AND rated above 4.5
SELECT product_name, price, rating
FROM   products
WHERE  category = 'audio'
  AND  stock_qty > 0
  AND  rating > 4.5;
```

`OR` returns rows that satisfy *at least one* condition:

```sql
-- Cheap products OR highly rated products
SELECT product_name, price, rating
FROM   products
WHERE  price < 60
   OR  rating >= 4.7;
```

`NOT` inverts a condition:

```sql
-- Products not in the audio category
SELECT product_name, category
FROM   products
WHERE  NOT category = 'audio';
-- Equivalent to: WHERE category != 'audio'
```

---

## Operator Precedence — Use Parentheses

`AND` has higher precedence than `OR`. This is the most dangerous pitfall in WHERE clauses:

```sql
-- AMBIGUOUS — what does this return?
WHERE category = 'audio' OR category = 'keyboard' AND price < 80

-- SQL reads it as:
WHERE category = 'audio' OR (category = 'keyboard' AND price < 80)
-- Returns ALL audio products + keyboards under $80
-- NOT "(audio or keyboard) products under $80"

-- If you meant the second interpretation, you MUST add parentheses:
WHERE (category = 'audio' OR category = 'keyboard') AND price < 80
```

**Rule of thumb:** whenever you mix `AND` and `OR`, add parentheses to make your intent explicit.
Future you will thank present you.

---

## Putting It All Together — A Realistic Query

```sql
-- Find in-stock products in the audio or monitor category,
-- priced between $50 and $400, with a known rating above 4.4
SELECT
    product_name,
    category,
    price,
    rating
FROM   products
WHERE  category      IN ('audio', 'monitor')
  AND  stock_qty     > 0
  AND  price         BETWEEN 50 AND 400
  AND  rating        IS NOT NULL
  AND  rating        > 4.4
ORDER  BY price;
```

Result: JBL Clip 4 ($59.99, 4.5), Sony WH-1000XM5 ($279.99, 4.8), Dell Monitor ($349.99, 4.6).

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  WHERE AND FILTERING — KEY TAKEAWAYS                                 │
├─────────────────────────────┬────────────────────────────────────────┤
│  =, !=, >, <, >=, <=        │  Basic comparison operators           │
│  BETWEEN a AND b            │  Inclusive range (works on dates too) │
│  IN (val1, val2, ...)       │  Match any value in a list            │
│  NOT IN (...)               │  Exclude values (watch for NULLs!)    │
│  LIKE 'pattern%'            │  Text pattern, % = wildcard           │
│  ILIKE (PostgreSQL)         │  Case-insensitive LIKE                │
│  IS NULL / IS NOT NULL      │  ONLY way to check for NULL           │
│  = NULL                     │  NEVER use this — always wrong        │
│  AND vs OR precedence       │  AND binds tighter — use parentheses  │
└─────────────────────────────┴────────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← SELECT and FROM](./select_and_from.md) &nbsp;|&nbsp; **Next:** [ORDER BY and LIMIT →](./order_by_and_limit.md)

**Related Topics:** [SELECT and FROM](./select_and_from.md) · [ORDER BY and LIMIT](./order_by_and_limit.md) · [DISTINCT and Aliases](./distinct_and_aliases.md)

---

## 📝 Practice Questions

- 📝 [Q4 · where-filtering](../sql_practice_questions_100.md#q4--normal--where-filtering)
- 📝 [Q5 · between-in-like](../sql_practice_questions_100.md#q5--normal--between-in-like)
- 📝 [Q6 · is-null](../sql_practice_questions_100.md#q6--critical--is-null)
- 📝 [Q7 · null-handling](../sql_practice_questions_100.md#q7--critical--null-handling)
- 📝 [Q8 · and-or-not](../sql_practice_questions_100.md#q8--thinking--and-or-not)

