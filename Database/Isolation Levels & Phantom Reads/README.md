# Isolation Levels & Phantom Reads — PostgreSQL Deep Dive

## 🧩 What Is a Transaction Isolation Level?

Isolation levels define how and when the changes made by one transaction become visible to others. They control concurrency side effects such as:

* **Dirty Reads** – Reading uncommitted data.
* **Non-repeatable Reads** – Reading the same row twice gives different results.
* **Phantom Reads** – New rows appear/disappear when re-running a range query.

PostgreSQL uses **MVCC (Multi-Version Concurrency Control)** to achieve isolation without locking reads.

---

## 🧱 PostgreSQL Isolation Levels

### 1. **READ UNCOMMITTED** (Not truly supported)

* PostgreSQL treats it **as READ COMMITTED**.
* No dirty reads are allowed because each query sees only committed data.

📘 **Effect:**

* ✅ Prevents **Dirty Reads**
* ❌ Allows **Non-repeatable Reads**
* ❌ Allows **Phantom Reads**

📊 **Example:**

```sql
-- Transaction A
BEGIN;
SELECT COUNT(*) FROM orders WHERE total > 100;
-- Returns 5

-- Transaction B inserts a new matching row and commits
INSERT INTO orders VALUES (...);
COMMIT;

-- Transaction A re-runs the same query
SELECT COUNT(*) FROM orders WHERE total > 100;
-- Returns 6 (phantom read)
COMMIT;
```

---

### 2. **READ COMMITTED** (PostgreSQL’s Default)

Each query in the same transaction sees data **committed before the query began**. If another transaction commits new data between two queries, the second query can see it.

📘 **Effect:**

* ✅ Prevents **Dirty Reads** (only sees committed data)
* ❌ Allows **Non-repeatable Reads**
* ❌ Allows **Phantom Reads**

📊 **Example:**

```sql
-- Transaction A
BEGIN;
SELECT * FROM products WHERE category = 'Books'; -- sees 10 rows

-- Transaction B
INSERT INTO products VALUES (... 'Books' ...);
COMMIT;

-- Transaction A runs the same query again
SELECT * FROM products WHERE category = 'Books'; -- now sees 11 rows
COMMIT;
```

Here, a **phantom row** appeared between the two queries.

💡 PostgreSQL prevents dirty reads but not phantoms here because each query gets a fresh snapshot at execution time.

---

### 3. **REPEATABLE READ**

All queries in a transaction see a **snapshot of the database taken when the transaction began**. Even if other transactions commit new data, the current transaction won’t see those changes.

📘 **Effect:**

* ✅ Prevents **Dirty Reads**
* ✅ Prevents **Non-repeatable Reads**
* ✅ Prevents **Phantom Reads** (under MVCC)

📊 **Example:**

```sql
-- Transaction A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM orders WHERE status = 'pending'; -- returns 10

-- Transaction B inserts a new 'pending' order and commits
INSERT INTO orders (status) VALUES ('pending');
COMMIT;

-- Transaction A runs the same query again
SELECT COUNT(*) FROM orders WHERE status = 'pending'; -- still returns 10
COMMIT;
```

Even though a new row was added, Transaction A doesn’t see it.

💡 **Why?** PostgreSQL uses MVCC snapshots. Transaction A keeps using the same snapshot, so no new rows appear → no phantom reads.

---

### 4. **SERIALIZABLE** (Strongest Level)

PostgreSQL implements **Serializable Snapshot Isolation (SSI)** — a combination of MVCC + validation to ensure the result is equivalent to some serial order of transactions.

📘 **Effect:**

* ✅ Prevents **Dirty Reads**
* ✅ Prevents **Non-repeatable Reads**
* ✅ Prevents **Phantom Reads**
* ✅ Ensures full **serializability**

📊 **Example:**

```sql
-- Transaction A
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts WHERE region = 'North';

-- Transaction B adds a new account in the same region
INSERT INTO accounts (region, balance) VALUES ('North', 500);
COMMIT;

-- Transaction A tries to run again
SELECT SUM(balance) FROM accounts WHERE region = 'North';
-- PostgreSQL may raise a serialization failure:
-- ERROR: could not serialize access due to concurrent update
ROLLBACK;
```

Instead of allowing a phantom, PostgreSQL **forces a rollback** to maintain a serial order.

💡 **How PostgreSQL prevents phantom reads:**

* In **READ COMMITTED**, each statement sees the latest committed state (phantoms possible).
* In **REPEATABLE READ**, all statements use a single snapshot, so phantoms are invisible.
* In **SERIALIZABLE**, if a phantom could occur (changing logical result), PostgreSQL aborts the transaction.

---

## 🔬 How PostgreSQL Prevents Phantom Reads Internally

PostgreSQL uses **MVCC (Multi-Version Concurrency Control)** to avoid blocking reads:

* Each row version has **xmin** (creator transaction ID) and **xmax** (deleter transaction ID).
* When a transaction starts, it captures a **snapshot** — a list of active transaction IDs.
* Queries only see rows where `xmin` is committed and `xmax` is not yet visible.

In **REPEATABLE READ**, the same snapshot is reused throughout the transaction, so new inserts (possible phantoms) are invisible.
In **SERIALIZABLE**, the system tracks **conflicting read-write dependencies** between transactions and rolls back one if serializability could be broken.

---

## 🧠 Summary Table

| Isolation Level  | Dirty Read | Non-Repeatable Read | Phantom Read | Notes                                   |
| ---------------- | ---------- | ------------------- | ------------ | --------------------------------------- |
| Read Uncommitted | ❌          | ✅                   | ✅            | Treated as Read Committed               |
| Read Committed   | ❌          | ✅                   | ✅            | Default; each query gets fresh snapshot |
| Repeatable Read  | ❌          | ❌                   | ❌            | Snapshot fixed for transaction          |
| Serializable     | ❌          | ❌                   | ❌            | Full serializable guarantees            |

---

## 🏁 Final Thoughts

* PostgreSQL’s **MVCC** makes reads non-blocking while maintaining isolation.
* **REPEATABLE READ** in PostgreSQL is strong enough to prevent phantom reads, unlike in some other databases (like MySQL).
* **SERIALIZABLE** ensures perfect serial execution order by detecting and aborting unsafe transactions.
