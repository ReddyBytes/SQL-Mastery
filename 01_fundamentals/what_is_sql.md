# What is SQL?

## The Library Analogy

Picture a vast library with millions of books — novels, textbooks, journals, newspapers — all
organised on shelves behind the counter. You walk up and tell the librarian:

> "I'd like every mystery novel published after 2015, sorted by author's last name, please."

The librarian understands your request, disappears into the stacks, and returns with exactly those
books. You didn't rearrange the shelves yourself. You didn't walk into the back room. You spoke a
request in a language the librarian understood, and the librarian did the work.

**SQL is that language.** Instead of books, the library holds data. Instead of a human librarian,
a database engine processes your request. And instead of plain English, you write structured
queries — but they read surprisingly close to plain English once you know the vocabulary.

---

## What Does SQL Stand For?

**SQL = Structured Query Language** (pronounced either "S-Q-L" or "sequel" — both are acceptable).

The key word is *query*. SQL is fundamentally a language for **asking questions about data** and
**instructing a system to change data**. It covers four main operations, often called **CRUD**:

```
  CREATE / INSERT  →  Add new data
  READ   / SELECT  →  Ask questions, retrieve data
  UPDATE           →  Change existing data
  DELETE           →  Remove data
```

---

## Your First SQL Statement — Decoded

Here is a real SQL query. Read it out loud as if it were English:

```sql
SELECT first_name, email
FROM   customers
WHERE  country = 'Canada';
```

Now translate it word by word:

```
SELECT first_name, email   →  "Give me the first_name and email columns"
FROM   customers           →  "...from the table called customers"
WHERE  country = 'Canada'  →  "...but only rows where the country is Canada"
;                          →  "End of request."
```

That's it. SQL is intentionally readable. A non-programmer can look at that query and understand
what it does. That readability is one reason SQL has survived for over 50 years.

---

## Where Is SQL Used?

SQL is everywhere data lives. Some concrete examples:

```
  Industry          Use case
  ──────────────────────────────────────────────────────────────
  Web applications  Store user accounts, sessions, purchases
  Analytics / BI    Calculate revenue, user retention, funnels
  Finance           Transaction ledgers, risk calculations
  Healthcare        Patient records, lab results, prescriptions
  Gaming            Leaderboards, player inventories, matchmaking
  Logistics         Inventory, shipments, delivery tracking
  Data Science      Feature engineering, model training datasets
```

When a developer at Spotify queries "which songs did this user play most this month?", that's SQL.
When an accountant exports last quarter's revenue by region, that's SQL. When your bank flags a
suspicious transaction, SQL is almost certainly involved.

---

## A Brief History — Why Has SQL Survived 50 Years?

```
  Year    Event
  ──────────────────────────────────────────────────────────────────────────
  1970    Edgar F. Codd (IBM) publishes relational model theory
  1974    IBM researchers create SEQUEL (Structured English Query Language)
  1979    Oracle ships the first commercial SQL database
  1986    ANSI publishes the first SQL standard (SQL-86)
  1992    SQL-92 — still the baseline most systems follow today
  1999    SQL:1999 adds recursion, triggers, procedural extensions
  2003    SQL:2003 adds window functions (one of the most powerful features)
  2011    SQL:2011 adds temporal data support
  Today   PostgreSQL, MySQL, SQLite, SQL Server, BigQuery — all speak SQL
```

SQL outlasted dozens of competitors because of three properties:

1. **Declarative** — you describe *what* you want, not *how* to get it. The database engine
   figures out the most efficient execution plan automatically.

2. **Set-based** — SQL operates on entire sets of rows at once, not one row at a time. This
   makes it naturally parallel and fast on modern hardware.

3. **Standardised** — code you write for PostgreSQL is 80–90% compatible with MySQL, SQLite,
   and SQL Server. Skills transfer across employers and projects.

---

## The Four Categories of SQL Commands

SQL statements fall into four groups. You'll use SELECT by far the most:

```
  Category    Full name                 Example commands
  ──────────────────────────────────────────────────────────────────
  DQL         Data Query Language       SELECT
  DML         Data Manipulation Lang.   INSERT, UPDATE, DELETE
  DDL         Data Definition Lang.     CREATE TABLE, ALTER, DROP
  DCL         Data Control Lang.        GRANT, REVOKE
```

Most of this course focuses on **DQL** (reading data) and **DML** (modifying data), because those
are the skills used every day by analysts, engineers, and developers.

---

## SQL Is Not a Programming Language (Kind Of)

SQL is often called a *query language*, not a programming language, because it lacks traditional
constructs like loops and variables in its core form. However, every major database adds
procedural extensions:

- PostgreSQL adds **PL/pgSQL** (stored procedures, functions)
- MySQL adds **stored procedures** with its own syntax
- SQL Server adds **T-SQL** (Transact-SQL)

For most of this course, we stay in standard SQL — the parts that work everywhere. Procedural
extensions appear later in the advanced sections.

---

## Setting Up a Practice Environment

You don't need to install anything to start learning. Options from easiest to most production-like:

```
  Option                    Notes
  ──────────────────────────────────────────────────────────────────
  db-fiddle.com             Browser-based, supports PostgreSQL/MySQL
  sqlfiddle.com             Similar, very beginner-friendly
  DBeaver (free app)        Connect to any database, great UI
  psql (terminal)           PostgreSQL's built-in CLI — what pros use
  Docker + PostgreSQL       Full local environment in minutes
```

Throughout this course, examples use **PostgreSQL syntax**. Where MySQL or SQLite differ
meaningfully, a callout box will highlight the difference.

---

## A Taste of What's Coming

```sql
-- Find the top 5 customers by total spending this year
SELECT
    c.first_name || ' ' || c.last_name  AS customer_name,
    SUM(o.total_amount)                 AS total_spent
FROM   customers c
JOIN   orders    o ON c.id = o.customer_id
WHERE  o.created_at >= '2024-01-01'
GROUP  BY c.id, c.first_name, c.last_name
ORDER  BY total_spent DESC
LIMIT  5;
```

Every keyword in that query will be second nature to you by the end of this course. Right now,
just notice that it still reads almost like English: "Select the customer name and sum of orders,
from customers joined with orders, where the order was in 2024, grouped by customer, ordered by
spending, top 5 only."

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  WHAT IS SQL — KEY TAKEAWAYS                                         │
├──────────────────────────────┬───────────────────────────────────────┤
│  SQL stands for              │  Structured Query Language            │
│  Created                     │  IBM, early 1970s; ANSI std 1986      │
│  Core purpose                │  Read and manipulate relational data  │
│  Why it survived             │  Declarative, set-based, standardised │
│  Your most-used command      │  SELECT                               │
│  4 command categories        │  DQL, DML, DDL, DCL                   │
│  PostgreSQL extension        │  PL/pgSQL for procedural logic        │
│  Quick practice tool         │  db-fiddle.com                        │
└──────────────────────────────┴───────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** — (this is the first file) &nbsp;|&nbsp; **Next:** [Databases and RDBMS →](./databases_and_rdbms.md)

**Related Topics:** [Databases and RDBMS](./databases_and_rdbms.md) · [SQL vs NoSQL](./sql_vs_nosql.md)
