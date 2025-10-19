# Database Isolation Levels & Phantom Reads

## 🧩 Overview

Database **isolation levels** determine how transactions interact with each other in concurrent environments. They control visibility of changes made by one transaction to others before those changes are committed.

In relational databases, isolation levels are defined by the **ANSI SQL standard**, and they exist to balance between **data consistency** and **performance**.

---

## 🔒 Isolation Levels (From Weakest to Strongest)

| Level                | Dirty Reads | Non-Repeatable Reads | Phantom Reads |
| -------------------- | ----------- | -------------------- | ------------- |
| **Read Uncommitted** | ✅ Possible  | ✅ Possible           | ✅ Possible    |
| **Read Committed**   | ❌ Prevented | ✅ Possible           | ✅ Possible    |
| **Repeatable Read**  | ❌ Prevented | ❌ Prevented          | ✅ Possible    |
| **Serializable**     | ❌ Prevented | ❌ Prevented          | ❌ Prevented   |

---

## 🧠 Key Concepts

### 1. Dirty Read

Occurs when a transaction reads uncommitted data from another transaction.

**Example:**

```sql
-- Transaction A
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- (not committed yet)

-- Transaction B
SELECT balance FROM accounts WHERE id = 1;  -- sees the uncommitted value
```

### 2. Non-Repeatable Read

Occurs when a transaction reads the same row twice and gets different data because another transaction modified and committed it in between.

**Example:**

```sql
-- Transaction A
SELECT balance FROM accounts WHERE id = 1;  -- balance = 1000

-- Transaction B
UPDATE accounts SET balance = 800 WHERE id = 1;
COMMIT;

-- Transaction A
SELECT balance FROM accounts WHERE id = 1;  -- balance = 800 (changed)
```

### 3. Phantom Read 🧙‍♂️

Occurs when a transaction re-executes a query and finds **new rows** (or missing rows) that match the search condition due to another transaction inserting or deleting data.

**Example:**

```sql
-- Transaction A
SELECT * FROM orders WHERE amount > 100;  -- returns 5 rows

-- Transaction B
INSERT INTO orders (id, amount) VALUES (6, 150);
COMMIT;

-- Transaction A re-executes the same query
SELECT * FROM orders WHERE amount > 100;  -- now returns 6 rows (phantom row appeared)
```

---

## 🌈 Types of Phantom Reads (Detailed)

Phantom reads can occur in several subtle ways depending on what data is inserted, deleted, or updated.

### 🪄 1. **Insert Phantom**

A new row is added by another transaction and now matches the query condition.

```sql
-- Transaction A
SELECT * FROM employees WHERE salary > 5000;  -- 10 rows

-- Transaction B
INSERT INTO employees (id, name, salary) VALUES (11, 'John', 6000);
COMMIT;

-- Transaction A
SELECT * FROM employees WHERE salary > 5000;  -- now 11 rows
```

### 💨 2. **Delete Phantom**

A row matching the condition is deleted by another transaction.

```sql
-- Transaction A
SELECT * FROM orders WHERE status = 'pending';  -- 4 rows

-- Transaction B
DELETE FROM orders WHERE id = 2 AND status = 'pending';
COMMIT;

-- Transaction A
SELECT * FROM orders WHERE status = 'pending';  -- now 3 rows
```

### 🧾 3. **Update Phantom**

A row that previously didn’t match the condition is updated so it now does.

```sql
-- Transaction A
SELECT * FROM products WHERE price > 100;  -- 5 rows

-- Transaction B
UPDATE products SET price = 150 WHERE id = 10;
COMMIT;

-- Transaction A
SELECT * FROM products WHERE price > 100;  -- now 6 rows
```

---

## 🧮 Isolation Level Behaviors in Practice

| Isolation Level      | Dirty Read | Non-Repeatable Read | Phantom Read | Example Database                             |
| -------------------- | ---------- | ------------------- | ------------ | -------------------------------------------- |
| **Read Uncommitted** | ✅          | ✅                   | ✅            | Rare (performance focus)                     |
| **Read Committed**   | ❌          | ✅                   | ✅            | SQL Server (default), Oracle                 |
| **Repeatable Read**  | ❌          | ❌                   | ✅            | MySQL (InnoDB default)                       |
| **Serializable**     | ❌          | ❌                   | ❌            | PostgreSQL (Serializable Snapshot Isolation) |

---

## 🧩 Preventing Phantom Reads

To prevent phantom reads:

1. Use **Serializable isolation level**.
2. Use **predicate locking** — locks that cover ranges of rows that match a condition.
3. Consider **optimistic concurrency control** in high-performance systems.

**Example (Serializable):**

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;

SELECT * FROM orders WHERE amount > 100;
-- Another transaction trying to insert a matching order will be blocked until commit

COMMIT;
```

---

## 🏁 Summary

| Problem             | Cause                                    | Solved At Level |
| ------------------- | ---------------------------------------- | --------------- |
| Dirty Read          | Uncommitted read                         | Read Committed  |
| Non-Repeatable Read | Committed updates                        | Repeatable Read |
| Phantom Read        | Insert/Delete/Update changing result set | Serializable    |

---

> 💡 **Tip:** In most modern systems (e.g., MySQL’s InnoDB or PostgreSQL), **Repeatable Read** is often enough for consistency, but **Serializable** ensures perfect isolation at a performance cost.
