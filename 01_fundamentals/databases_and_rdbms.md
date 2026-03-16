# Databases and RDBMS

## The Spreadsheet-on-Steroids Analogy

You've probably built a spreadsheet to track something — maybe a list of contacts, a budget, or
a project task list. A single spreadsheet tab works fine at first: one row per person, columns for
name, email, phone number, city.

Then things get complicated. You add a second tab for orders. Now you're copy-pasting customer
names across tabs. Someone spells "Catherine" as "Kathryn" in the orders tab. Totals break.
The file grows to 80,000 rows and Excel grinds to a halt.

A **relational database** is what you reach for when spreadsheets stop being enough. Think of it
as a collection of linked spreadsheet tabs (called **tables**), where:

- Every tab has strictly enforced column types (no mixing text and numbers in the same column)
- Rows in one tab can *point to* rows in another tab via a shared ID — no copy-pasting
- The system enforces rules automatically: you can't add an order for a customer who doesn't exist
- Millions of rows are handled in milliseconds using indexes
- Multiple users can read and write simultaneously without corrupting data

---

## What Exactly Is a Database?

A **database** is an organised collection of data stored on disk (or in memory), managed by
software that controls how data is stored, retrieved, and protected.

A **Database Management System (DBMS)** is that software — the engine that sits between your
application and the raw data files on disk.

A **Relational DBMS (RDBMS)** specifically organises data into **tables** (relations in academic
terminology) and uses SQL as its query language.

---

## Tables, Rows, and Columns

Every piece of data in an RDBMS lives inside a **table**. A table is exactly like a spreadsheet
tab: it has named columns (the structure) and rows (the data).

```
  TABLE: customers
  ┌─────┬────────────┬───────────┬─────────────────────────┬─────────┐
  │ id  │ first_name │ last_name │ email                   │ country │
  ├─────┼────────────┼───────────┼─────────────────────────┼─────────┤
  │  1  │ Alice      │ Nguyen    │ alice@example.com       │ Canada  │
  │  2  │ Bob        │ Martinez  │ bob@example.com         │ USA     │
  │  3  │ Chen       │ Wei       │ chen.wei@example.com    │ China   │
  │  4  │ Diana      │ Okafor    │ diana.o@example.com     │ Nigeria │
  └─────┴────────────┴───────────┴─────────────────────────┴─────────┘
        ↑
     Each row is one customer.
     Each column is one attribute (piece of information) about that customer.
```

Formal vocabulary:
- **Table** = a named set of rows and columns (also called a *relation*)
- **Row** = one record (also called a *tuple*)
- **Column** = one attribute, with a specific data type (also called a *field*)
- **Schema** = the overall structure — which tables exist and what columns each has

---

## Primary Keys — Every Row Needs a Unique ID

A **primary key** is a column (or combination of columns) whose value uniquely identifies every
row in the table. No two rows can have the same primary key, and it can never be NULL.

```
  TABLE: customers
  ┌──────┬────────────┬───────────┐
  │  id  │ first_name │ last_name │    ← id is the PRIMARY KEY
  ├──────┼────────────┼───────────┤      No two customers share an id.
  │  1   │ Alice      │ Nguyen    │      The database enforces this.
  │  2   │ Bob        │ Martinez  │
  │  3   │ Chen       │ Wei       │
  └──────┴────────────┴───────────┘
```

Common primary key strategies:

```
  Strategy          Example         Notes
  ──────────────────────────────────────────────────────────────────
  Auto-increment    1, 2, 3, 4...   Simple, common (SERIAL in PG)
  UUID              a3f9-bc12-...   Globally unique, good for APIs
  Natural key       email address   Risky — people change emails
```

In PostgreSQL you'd declare it like this:

```sql
CREATE TABLE customers (
    id         SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name  VARCHAR(100) NOT NULL,
    email      VARCHAR(255) UNIQUE NOT NULL,
    country    VARCHAR(100)
);
```

---

## Foreign Keys — Linking Tables Together

A **foreign key** is a column in one table that refers to the primary key of another table. This
is how relationships work in an RDBMS — no data duplication, just references.

```
  TABLE: customers                  TABLE: orders
  ┌─────┬────────────┐              ┌─────┬─────────────┬──────────────┬─────────────┐
  │ id  │ first_name │              │ id  │ customer_id │ total_amount │ created_at  │
  ├─────┼────────────┤              ├─────┼─────────────┼──────────────┼─────────────┤
  │  1  │ Alice      │◄─────────────│  1  │      1      │    149.99    │ 2024-03-01  │
  │  2  │ Bob        │◄─────────┐   │  2  │      1      │     49.50    │ 2024-03-15  │
  │  3  │ Chen       │          └───│  3  │      2      │    220.00    │ 2024-04-02  │
  └─────┴────────────┘              └─────┴─────────────┴──────────────┴─────────────┘
                                           ↑
                                    FOREIGN KEY → customers.id
                                    Alice (id=1) has 2 orders.
                                    Bob (id=2) has 1 order.
```

The database **enforces** this link. If you try to insert an order with `customer_id = 99` but
no customer with id 99 exists, the database will reject it. This is called **referential
integrity** — one of the core strengths of an RDBMS.

```sql
CREATE TABLE orders (
    id            SERIAL PRIMARY KEY,
    customer_id   INT NOT NULL REFERENCES customers(id),
    total_amount  NUMERIC(10, 2) NOT NULL,
    created_at    TIMESTAMPTZ DEFAULT NOW()
);
```

---

## The ACID Guarantee

RDBMS systems provide **ACID** guarantees — four properties that make relational databases
reliable for critical data:

```
  Property      Meaning
  ──────────────────────────────────────────────────────────────────────────
  Atomicity     A transaction either fully succeeds or fully fails.
                If your bank transfer deducts $100 but the credit fails,
                the deduction is rolled back automatically.

  Consistency   Every transaction brings the database from one valid state
                to another. Rules (constraints) are always enforced.

  Isolation     Concurrent transactions don't interfere with each other.
                Two people booking the last seat on a flight can't both win.

  Durability    Once committed, data survives crashes, power cuts, etc.
                The database writes to disk before confirming success.
```

---

## Popular RDBMS Systems — When to Use Which

```
  System          Best for                              Notes
  ──────────────────────────────────────────────────────────────────────────
  PostgreSQL      Production web apps, analytics        Open source, feature-rich
                  complex queries, JSON support         This course's primary focus

  MySQL /         Web apps, WordPress/Drupal,           Slightly simpler, huge
  MariaDB         read-heavy workloads                  community, free

  SQLite          Mobile apps, desktop apps,            Zero-config, single file,
                  testing, embedded systems             not for concurrent writes

  SQL Server      Enterprise Windows environments,      Microsoft ecosystem,
                  .NET applications                     excellent tooling

  Oracle DB       Large enterprise, banking,            Very powerful, very expensive,
                  government systems                    used in legacy systems

  Amazon RDS /    Cloud-hosted production databases     Managed — no server admin
  Aurora          (PostgreSQL and MySQL compatible)
```

> **Syntax note:** Most SQL in this course is standard PostgreSQL. Where MySQL or SQLite differ:
> - MySQL: `AUTO_INCREMENT` instead of `SERIAL`; backticks instead of double-quotes for names
> - SQLite: very permissive types, no `SERIAL` (use `INTEGER PRIMARY KEY` instead)

---

## A Mini Schema in Action

Here's a small but realistic schema — three tables, two relationships:

```
  customers ──< orders >── products
                  │
                  └── order_items (junction table)

  customers: id, first_name, last_name, email, country
  products:  id, name, price, stock_quantity
  orders:    id, customer_id, created_at, status
  order_items: order_id, product_id, quantity, unit_price
```

This pattern appears in virtually every e-commerce system. Understanding it will let you read
production database schemas from day one.

---

## Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  DATABASES AND RDBMS — KEY TAKEAWAYS                                 │
├────────────────────────────────┬─────────────────────────────────────┤
│  Database                      │  Organised, managed data store      │
│  RDBMS                         │  Uses tables, rows, columns, SQL    │
│  Table                         │  Named set of rows + columns        │
│  Primary key                   │  Unique identifier for each row     │
│  Foreign key                   │  Points to a row in another table   │
│  Referential integrity         │  DB enforces valid FK references    │
│  ACID                          │  Atomic, Consistent, Isolated,      │
│                                │  Durable — the reliability promise  │
│  Best open-source RDBMS        │  PostgreSQL                         │
│  Best embedded / testing       │  SQLite                             │
└────────────────────────────────┴─────────────────────────────────────┘
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← What is SQL](./what_is_sql.md) &nbsp;|&nbsp; **Next:** [SQL vs NoSQL →](./sql_vs_nosql.md)

**Related Topics:** [What is SQL](./what_is_sql.md) · [SQL vs NoSQL](./sql_vs_nosql.md)
