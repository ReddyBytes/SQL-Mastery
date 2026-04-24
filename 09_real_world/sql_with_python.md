# SQL with Python — Connecting Your Code to Your Database

## The Translator Analogy

SQL is the language your database speaks fluently. Python is your general-purpose assistant who can do everything else — send emails, process files, build APIs, run machine learning models. The two need to talk to each other.

Python acts as the translator: it sends SQL to the database in the language the database understands, receives the results, and hands them back to you as Python objects (lists, dictionaries, DataFrames) that your code can work with.

There are a few translators to choose from: `psycopg2` (direct PostgreSQL driver), `sqlite3` (built-in, zero setup), and `SQLAlchemy` (ORM that works with many databases). Each has its place.

---

## sqlite3 — Zero Setup, Built Into Python

`sqlite3` is in the Python standard library. Great for learning, prototyping, and small tools.

```python
import sqlite3

# Connect (creates the file if it doesn't exist)
conn = sqlite3.connect('shop.db')
cursor = conn.cursor()

# Create a table
cursor.execute("""
    CREATE TABLE IF NOT EXISTS products (
        product_id   INTEGER PRIMARY KEY AUTOINCREMENT,
        name         TEXT    NOT NULL,
        category     TEXT    NOT NULL,
        price        REAL    NOT NULL,
        stock_qty    INTEGER NOT NULL DEFAULT 0
    )
""")
conn.commit()

# Insert a row safely (parameterised — never use string formatting!)
cursor.execute(
    "INSERT INTO products (name, category, price, stock_qty) VALUES (?, ?, ?, ?)",
    ('Wireless Mouse', 'Electronics', 29.99, 150)
)
conn.commit()

# Query and fetch results
cursor.execute("SELECT product_id, name, price FROM products WHERE category = ?",
               ('Electronics',))
rows = cursor.fetchall()
for row in rows:
    print(f"ID: {row[0]}, Name: {row[1]}, Price: £{row[2]:.2f}")

# Always close the connection
conn.close()
```

---

## psycopg2 — PostgreSQL from Python

`psycopg2` is the standard PostgreSQL adapter for Python. Install with `pip install psycopg2-binary`.

```python
import psycopg2
import psycopg2.extras  # for DictCursor

# Connect to PostgreSQL
conn = psycopg2.connect(
    host     = "localhost",
    port     = 5432,
    database = "shop",
    user     = "app_user",
    password = "s3cur3passw0rd"
)

# DictCursor lets you access columns by name (row['name']) not position (row[0])
cursor = conn.cursor(cursor_factory=psycopg2.extras.DictCursor)

# Query with parameters — use %s placeholders (NOT f-strings!)
cursor.execute("""
    SELECT user_id, name, email
    FROM   users
    WHERE  created_at > %s
      AND  is_active = %s
""", ('2024-01-01', True))

users = cursor.fetchall()
for user in users:
    print(f"{user['name']} <{user['email']}>")

cursor.close()
conn.close()
```

### Context Manager Pattern (Recommended)

```python
import psycopg2
from contextlib import contextmanager

@contextmanager
def get_db():
    conn = psycopg2.connect(
        host="localhost", database="shop",
        user="app_user", password="s3cur3passw0rd"
    )
    try:
        yield conn
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        conn.close()

# Usage
with get_db() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT COUNT(*) FROM orders WHERE status = %s", ('pending',))
        count = cur.fetchone()[0]
        print(f"Pending orders: {count}")
```

---

## SQL Injection — The Dangerous vs Safe Way

SQL injection is one of the most common security vulnerabilities. It happens when user input is concatenated directly into a SQL string.

```python
# DANGEROUS — NEVER DO THIS
user_input = "alice@example.com'; DROP TABLE users; --"

query = f"SELECT * FROM users WHERE email = '{user_input}'"
# Resulting SQL:
# SELECT * FROM users WHERE email = 'alice@example.com'; DROP TABLE users; --'
# The attacker just dropped your users table!

cursor.execute(query)  # catastrophic
```

```python
# SAFE — Always use parameterised queries
user_input = "alice@example.com'; DROP TABLE users; --"

cursor.execute(
    "SELECT * FROM users WHERE email = %s",
    (user_input,)   # psycopg2 escapes this safely
)
# The driver treats the entire string as a value, not SQL
# The query is safe regardless of what user_input contains
```

```
+----------------------------+--------------------------------------------+
| Unsafe                     | Safe                                       |
+----------------------------+--------------------------------------------+
| f"... WHERE id = {uid}"    | "... WHERE id = %s", (uid,)               |
| "... WHERE id = " + str(x) | "... WHERE id = %s", (x,)                 |
| .format() with SQL         | Parameterised via driver                   |
+----------------------------+--------------------------------------------+
The rule: user data goes in parameters, NEVER in the query string itself.
```

---

## Inserting Multiple Rows Efficiently

```python
import psycopg2.extras

products = [
    ('Mechanical Keyboard', 'Electronics', 89.99, 75),
    ('Desk Lamp',           'Office',      19.99, 200),
    ('USB-C Hub',           'Electronics', 49.99, 60),
]

with get_db() as conn:
    with conn.cursor() as cur:
        psycopg2.extras.execute_values(
            cur,
            "INSERT INTO products (name, category, price, stock_qty) VALUES %s",
            products
        )
print(f"Inserted {len(products)} products")
```

---

## SQLAlchemy — ORM Basics

SQLAlchemy lets you interact with the database using Python objects instead of raw SQL. It supports PostgreSQL, MySQL, SQLite, and more.

```
pip install sqlalchemy psycopg2-binary
```

```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import Session

# Connection string
engine = create_engine(
    "postgresql+psycopg2://app_user:s3cur3passw0rd@localhost/shop",
    echo=False
)

# Run raw SQL through SQLAlchemy
with Session(engine) as session:
    result = session.execute(
        text("SELECT user_id, name FROM users WHERE is_active = :active"),
        {"active": True}
    )
    for row in result:
        print(row.user_id, row.name)
```

### ORM-style (Declarative)

```python
from sqlalchemy import Column, Integer, String, Numeric, Boolean
from sqlalchemy.orm import declarative_base, Session

Base = declarative_base()

class Product(Base):
    __tablename__ = "products"
    product_id = Column(Integer, primary_key=True)
    name       = Column(String, nullable=False)
    category   = Column(String, nullable=False)
    price      = Column(Numeric(10, 2), nullable=False)
    is_active  = Column(Boolean, default=True)

with Session(engine) as session:
    # Query
    electronics = (
        session.query(Product)
        .filter(Product.category == "Electronics", Product.price < 50)
        .all()
    )
    for p in electronics:
        print(p.name, p.price)
```

---

## pandas read_sql — Pull Data Straight Into a DataFrame

For analytics and reporting, `pandas` can query a database and return a ready-to-use DataFrame.

```python
import pandas as pd
import psycopg2

conn = psycopg2.connect(
    host="localhost", database="shop",
    user="app_user", password="s3cur3passw0rd"
)

# Pull monthly sales into a DataFrame
query = """
    SELECT
        DATE_TRUNC('month', created_at)  AS month,
        COUNT(*)                          AS order_count,
        SUM(total_amount)                 AS revenue,
        AVG(total_amount)                 AS avg_order_value
    FROM   orders
    WHERE  status     = 'completed'
      AND  created_at >= '2024-01-01'
    GROUP  BY 1
    ORDER  BY 1
"""

df = pd.read_sql(query, conn)
conn.close()

print(df.head())
#         month  order_count    revenue  avg_order_value
# 0  2024-01-01         1243  98234.55            79.03
# 1  2024-02-01         1087  84132.10            77.40
# 2  2024-03-01         1412  112876.90            79.94
```

### Save Report to CSV

```python
# Process and export
df['month'] = pd.to_datetime(df['month'])
df['revenue'] = df['revenue'].astype(float)

# Month-over-month growth
df['mom_growth_pct'] = df['revenue'].pct_change() * 100

# Save to CSV
output_path = '/reports/monthly_sales_2024.csv'
df.to_csv(output_path, index=False, float_format='%.2f')
print(f"Report saved to {output_path}")
```

---

## Complete Script: Monthly Sales Report

```python
"""
monthly_sales_report.py
Pull monthly sales from PostgreSQL, compute growth, save to CSV.
"""

import pandas as pd
import psycopg2
from datetime import datetime

DB_CONFIG = {
    "host":     "localhost",
    "database": "shop",
    "user":     "app_user",
    "password": "s3cur3passw0rd",
}

def get_monthly_sales(year: int) -> pd.DataFrame:
    query = """
        SELECT
            DATE_TRUNC('month', created_at)::DATE AS month,
            COUNT(DISTINCT order_id)              AS orders,
            COUNT(DISTINCT user_id)               AS customers,
            SUM(total_amount)                     AS revenue
        FROM   orders
        WHERE  status = 'completed'
          AND  EXTRACT(YEAR FROM created_at) = %(year)s
        GROUP  BY 1
        ORDER  BY 1
    """
    with psycopg2.connect(**DB_CONFIG) as conn:
        return pd.read_sql(query, conn, params={"year": year})

def main():
    year = datetime.now().year
    df = get_monthly_sales(year)
    df["revenue"] = df["revenue"].astype(float)
    df["mom_growth_pct"] = df["revenue"].pct_change().mul(100).round(2)

    filename = f"monthly_sales_{year}.csv"
    df.to_csv(filename, index=False)
    print(f"Saved {len(df)} rows to {filename}")
    print(df.to_string(index=False))

if __name__ == "__main__":
    main()
```

---

## Summary

```
+-------------------+--------------------------------------------------+
| Tool              | Use Case                                         |
+-------------------+--------------------------------------------------+
| sqlite3           | Built-in; local dev, small scripts, prototyping  |
| psycopg2          | Production PostgreSQL access from Python          |
| SQLAlchemy        | ORM; multi-database support; larger applications  |
| pandas read_sql   | Pull query results directly into a DataFrame      |
+-------------------+--------------------------------------------------+

Security rule (non-negotiable):
  ALWAYS use parameterised queries: cursor.execute(sql, (param,))
  NEVER build SQL strings with f-strings or concatenation from user input

Placeholder syntax:
  psycopg2  -->  %s
  sqlite3   -->  ?
  SQLAlchemy text()  -->  :param_name
```

---

**[🏠 Back to README](../README.md)**

**Prev:** [← Triggers](./triggers.md) &nbsp;|&nbsp; **Next:** [SQL Interview Questions →](../99_interview_master/sql_questions.md)

**Related Topics:** [Views](./views.md) · [Stored Procedures](./stored_procedures.md) · [Transactions](../07_data_modification/transactions.md)

---

## 📝 Practice Questions

- 📝 [Q75 · sql-with-python](../sql_practice_questions_100.md#q75--thinking--sql-with-python)

