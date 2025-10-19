# Isolation Levels & Phantom Reads

## Overview
This document covers database isolation levels and phantom reads, which are important concepts in database transaction management.

## Isolation Levels

Isolation levels define the degree to which transactions are isolated from each other. The SQL standard defines four isolation levels:

1. **Read Uncommitted**
   - Lowest isolation level
   - Allows dirty reads, non-repeatable reads, and phantom reads
   - Transactions can see uncommitted changes from other transactions

2. **Read Committed**
   - Prevents dirty reads
   - Still allows non-repeatable reads and phantom reads
   - Transactions only see committed changes

3. **Repeatable Read**
   - Prevents dirty reads and non-repeatable reads
   - Still allows phantom reads
   - Ensures that if a row is read twice in the same transaction, it will have the same values

4. **Serializable**
   - Highest isolation level
   - Prevents dirty reads, non-repeatable reads, and phantom reads
   - Transactions are completely isolated from each other

## Phantom Reads

A phantom read occurs when:
- A transaction re-executes a query returning a set of rows that satisfy a search condition
- Finds that the set of rows has changed due to another recently-committed transaction

### Example
```sql
-- Transaction 1
BEGIN TRANSACTION;
SELECT * FROM users WHERE age > 18;
-- Returns 10 rows

-- Transaction 2 (executes between Transaction 1's queries)
BEGIN TRANSACTION;
INSERT INTO users (name, age) VALUES ('John', 25);
COMMIT;

-- Transaction 1 (continues)
SELECT * FROM users WHERE age > 18;
-- Returns 11 rows (phantom row appeared!)
COMMIT;
```

## Prevention

Phantom reads can be prevented by:
- Using **Serializable** isolation level
- Using appropriate locking mechanisms
- Using database-specific features (e.g., range locks, predicate locks)

## Trade-offs

Higher isolation levels provide more consistency but:
- Reduce concurrency
- Increase the chance of deadlocks
- May decrease performance

Lower isolation levels provide:
- Better performance
- Higher concurrency
- But less consistency guarantees

## Notes

- Different database systems may implement isolation levels differently
- The default isolation level varies by database system
- Choose the isolation level based on your application's consistency requirements vs. performance needs
