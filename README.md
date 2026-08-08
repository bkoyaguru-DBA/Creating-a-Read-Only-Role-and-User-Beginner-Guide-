# Creating-a-Read-Only-Role-and-User-Beginner-Guide-
This guide walks through creating a database, adding schemas and tables, and setting up a **read-only role** that a real user can be assigned to.


It also includes the errors you will likely hit along the way, and *why*
they happen. Understanding the errors is more useful than memorizing the
commands, because it teaches you how Postgres permissions actually work.

---

## 1. Core Concept First

In Postgres, permissions are **layered**. Access to a table is not just one
grant — it's a chain:

```
CONNECT to database
   → USAGE on schema
      → SELECT on table (or sequence)
```

If any link in that chain is missing, you get a `permission denied` error —
even if the other two are correctly set. Most beginner mistakes come from
granting only one or two links in this chain.

---

## 2. Create the Database

```sql
CREATE DATABASE myuser;
```

Connect to it:

```sql
\c myuser
```

---

## 3. Create Schemas

A schema is like a folder inside a database that groups related tables.

```sql
CREATE SCHEMA sales;
CREATE SCHEMA accounting;
```

---

## 4. Create Tables and Insert Sample Data

```sql
CREATE TABLE sales.employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    hire_date DATE DEFAULT CURRENT_DATE,
    salary NUMERIC(10, 2)
);

INSERT INTO sales.employees (first_name, last_name, hire_date, salary)
VALUES
    ('Alice', 'Smith', '2025-03-15', 75000.00),
    ('Bob', 'Johnson', '2025-06-01', 68000.50),
    ('Charlie', 'Brown', '2026-01-10', 82000.00);
```

```sql
CREATE TABLE accounting.products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price NUMERIC(10, 2),
    is_available BOOLEAN DEFAULT true
);

INSERT INTO accounting.products (product_name, price, is_available)
VALUES
    ('Wireless Mouse', 29.99, true),
    ('Mechanical Keyboard', 89.50, true),
    ('Gaming Monitor', 249.00, false),
    ('USB-C Cable', 12.99, true);
```

---

## 5. Create the Read-Only Role

A **role** here is not a person — it's a permission template. You attach
real users to it later. `NOLOGIN` means this role can't be used to log in
directly; it only exists to hold permissions.

```sql
CREATE ROLE readonly_role NOLOGIN;
```

Check it exists:

```sql
\du
```

---

## 6. Grant Permissions (Step by Step)

### 6.1 Allow connecting to the database

```sql
GRANT CONNECT ON DATABASE myuser TO readonly_role;
```

### 6.2 Allow using schemas

By default, `public` schema access is common, but **custom schemas like
`sales` and `accounting` need their own explicit grant**. This is the step
most beginners forget.

```sql
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT USAGE ON SCHEMA sales TO readonly_role;
GRANT USAGE ON SCHEMA accounting TO readonly_role;
```

### 6.3 Allow reading tables in each schema

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA accounting TO readonly_role;
```

### 6.4 Allow reading sequences (needed for SERIAL/auto-increment columns)

```sql
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO readonly_role;
```

### 6.5 Auto-apply SELECT to future tables

Without this, any **new** table created later will NOT be readable by
`readonly_role` automatically — you'd have to grant it manually every time.

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO readonly_role;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON SEQUENCES TO readonly_role;
```

> **Note:** `ALTER DEFAULT PRIVILEGES` only applies to *future* objects
> created by the same role that ran the command, in that schema. It does
> not retroactively cover existing tables — that's why step 6.3 is still
> needed separately.

---

## 6.6 What Actually Happened, Step by Step (Real Errors)

This is what really happened while setting this up, before the schema
grants were added. Seeing the raw errors — not just a summary — makes it
obvious *why* each grant was needed.

**Attempt 1: Only granted `public` schema access. Tried querying `sales`:**

```
myuser=> select count(*) from sales.employees;
ERROR:  permission denied for schema sales
LINE 1: select count(*) from sales.employees;
                             ^
```

**Same attempt, tried `accounting` too:**

```
myuser=> select count(*) from accounting.products;
ERROR:  permission denied for schema accounting
LINE 1: select count(*) from accounting.products;
                             ^
```

**Why:** `adam` could connect to the database (`CONNECT` was granted), but
had no `USAGE` grant on the `sales` or `accounting` schemas. Postgres
blocks you at the schema door before it even looks at the table.

**Fix applied:**

```sql
GRANT USAGE ON SCHEMA sales TO readonly_role;
GRANT USAGE ON SCHEMA accounting TO readonly_role;
```

**Attempt 2: Schema access granted, but no table-level SELECT yet:**

```
myuser=> select count(*) from accounting.products;
ERROR:  permission denied for table products
```

**Why:** `USAGE` on a schema only lets you "see into" it — it does not
give you permission to read any table inside. You still need a separate
`SELECT` grant on the table itself.

**Fix applied:**

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA accounting TO readonly_role;
```

**Attempt 3: Reconnected and queried again — this time it worked:**

```
myuser=> select count(*) from accounting.products;
 count
-------
     4
(1 row)
```

**Takeaway:** the error message changes as you fix each layer —
`permission denied for schema` → `permission denied for table` → success.
That progression is your checklist. If you see "schema" in the error, fix
`USAGE`. If you see "table", fix `SELECT`.

---

## 7. Create a Real User and Assign the Role

```sql
CREATE USER adam WITH PASSWORD 'postgres123';
GRANT readonly_role TO adam;
```

> Use a strong password in real environments. This example password is for
> local testing only.

---

## 8. Test the User

Log in as the new user:

```bash
psql -U adam -d myuser -W -p 5432
```

Run a query:

```sql
SELECT count(*) FROM sales.employees;
```

---

## 9. Common Errors and What They Actually Mean

| Error | Cause | Fix |
|---|---|---|
| `permission denied for schema sales` | User can connect to the DB, but has no USAGE grant on that schema | `GRANT USAGE ON SCHEMA sales TO readonly_role;` |
| `permission denied for table products` | User can access the schema, but has no SELECT grant on tables inside it | `GRANT SELECT ON ALL TABLES IN SCHEMA accounting TO readonly_role;` |
| Works on old tables, fails on new ones | `ALTER DEFAULT PRIVILEGES` was not set | Run the `ALTER DEFAULT PRIVILEGES` command in section 6.5 |

**Key lesson:** Postgres errors tell you exactly which layer is missing —
schema-level or table-level. Read the error message literally; it's not
vague.

---

## 10. Final Working Result

```
myuser=> SELECT count(*) FROM accounting.products;
 count
-------
     4
(1 row)
```

---

## 11. Quick Reference: Full Working Script

```sql
-- 1. Database & schemas
CREATE DATABASE myuser;
\c myuser
CREATE SCHEMA sales;
CREATE SCHEMA accounting;

-- 2. Role
CREATE ROLE readonly_role NOLOGIN;

-- 3. Grants
GRANT CONNECT ON DATABASE myuser TO readonly_role;

GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT USAGE ON SCHEMA sales TO readonly_role;
GRANT USAGE ON SCHEMA accounting TO readonly_role;

GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA sales TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA accounting TO readonly_role;

GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO readonly_role;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON TABLES TO readonly_role;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT ON SEQUENCES TO readonly_role;

-- 4. User
CREATE USER adam WITH PASSWORD 'postgres123';
GRANT readonly_role TO adam;
```

---

## 12. Things to Improve Before Using This in Production

Be direct with yourself about the gaps in this setup:

- **Password is in plain text in the script.** Use a secrets manager or
  prompt for password instead of hardcoding it.
- **`ALTER DEFAULT PRIVILEGES` only covers `public` schema here.** If new
  tables get created in `sales` or `accounting` later, you need the same
  `ALTER DEFAULT PRIVILEGES` command run for those schemas too.
- **No `REVOKE` step shown.** If this DB previously had broader public
  access (e.g. default `PUBLIC` role grants), you should audit and revoke
  unintended access separately.
- **Single flat role.** For larger systems, consider separate roles per
  schema (e.g. `sales_readonly`, `accounting_readonly`) instead of one
  role with access to everything.
