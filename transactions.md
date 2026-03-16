# Transactions

- **Transactions** group logically related SQL statements into a single unit of work. They ensure data integrity and consistency.
- Oracle treats changes to table data as a two-step process: changes are made, then made permanent with a `COMMIT` or undone with a `ROLLBACK`.
- **COMMIT**: Makes all changes in the current transaction permanent and releases locks.
- **ROLLBACK**: Undoes all changes since the last `COMMIT` or to a specified `SAVEPOINT`, releasing locks.
- **SAVEPOINT**: Marks a point within a transaction to which you can roll back, allowing partial undo of changes.
- Transactions begin with the first executable SQL statement after a `COMMIT`, `ROLLBACK`, or connection.
- All transactional locks are released after a `COMMIT` or `ROLLBACK`.

## Syntax Quick Reference

```sql
-- Commit the current transaction
COMMIT;

-- Rollback the current transaction
ROLLBACK;

-- Create a savepoint
SAVEPOINT savepoint_name;

-- Rollback to a savepoint
ROLLBACK TO SAVEPOINT savepoint_name;
```

## Best Practices

- Always use transactions for operations that modify data in multiple steps.
- Use `SAVEPOINT` for complex transactions to allow partial rollbacks.
- Regularly `COMMIT` to avoid holding locks for too long.
- Use `ROLLBACK` to undo unwanted changes before committing.
- Check for errors before committing to ensure data integrity.

## Example Workflow

```sql
BEGIN;
	-- Step 1: Update data
	UPDATE accounts SET balance = balance - 1000 WHERE acct_no = 'A1';
	SAVEPOINT after_withdrawal;
	-- Step 2: Deposit
	UPDATE accounts SET balance = balance + 1000 WHERE acct_no = 'A2';
	-- If error occurs, rollback to savepoint
	-- ROLLBACK TO SAVEPOINT after_withdrawal;
COMMIT;
```

---

---

## Notes
- Use transactions to group related changes and ensure data consistency.
- Always `COMMIT` after successful operations to make changes permanent.
- Use `ROLLBACK` to undo unwanted changes before committing.
- `SAVEPOINT` allows partial rollbacks within a transaction.
- All locks are released after a `COMMIT` or `ROLLBACK`.

---

**Tip:** Use transactions for multi-step operations. Review your changes before committing to keep your database consistent.

---

## Transaction Properties (ACID)

- **Atomicity**: All operations in a transaction are completed; if not, the transaction is aborted and changes are rolled back.
- **Consistency**: A transaction brings the database from one valid state to another, maintaining all rules and constraints.
- **Isolation**: Transactions are isolated from each other; intermediate results are not visible to other transactions.
- **Durability**: Once committed, changes are permanent, even in the event of a system failure.

---

## Isolation Levels (Overview)

Most databases support different isolation levels to control visibility of changes between transactions:
- **Read Committed** (default in Oracle): Each query sees only committed data.
- **Serializable**: Transactions are completely isolated; acts as if transactions run one after another.
- **Read Uncommitted** and **Repeatable Read**: Not supported in Oracle, but common in other DBMS.

Isolation levels affect concurrency and consistency. Use the default unless you have specific requirements.

---

## Error Handling in Transactions

In PL/SQL and procedural SQL, error handling is crucial to ensure data integrity. Use the `EXCEPTION` block to catch and handle errors during a transaction. If an error occurs, you can `ROLLBACK` to undo changes.

**Example:**
```sql
BEGIN
		-- Start transaction
		UPDATE accounts SET balance = balance - 1000 WHERE acct_no = 'A1';
		UPDATE accounts SET balance = balance + 1000 WHERE acct_no = 'A2';
		COMMIT;
EXCEPTION
		WHEN OTHERS THEN
				ROLLBACK;
				DBMS_OUTPUT.PUT_LINE('Error occurred. Transaction rolled back.');
END;
```

---

## Edge Cases in Transactions

- **Session Disconnect:** If a session disconnects before a `COMMIT`, all uncommitted changes are automatically rolled back by Oracle.
- **System Crash:** If the database crashes before a `COMMIT`, changes are not saved. Oracle uses redo logs to recover committed transactions and roll back uncommitted ones.
- **DDL Statements:** Executing DDL (like `CREATE`, `ALTER`, `DROP`) causes an implicit `COMMIT` before and after the statement.

---

## Advanced Isolation Level Anomalies

Isolation levels control visibility of changes between transactions. Lower isolation can lead to anomalies:

- **Dirty Read:** Transaction reads uncommitted changes from another transaction. (Not possible in Oracle; Oracle does not support Read Uncommitted.)
- **Non-Repeatable Read:** A row is read twice in the same transaction and gets different values because another transaction modified and committed it in between. (Avoided in Serializable isolation.)
- **Phantom Read:** A transaction re-executes a query returning a set of rows that satisfy a condition and finds that the set has changed due to another recently-committed transaction (e.g., new rows inserted). (Avoided in Serializable isolation.)

**Example:**
```sql
-- Transaction 1
BEGIN;
SELECT * FROM accounts WHERE balance > 1000;

-- Transaction 2 (in parallel)
INSERT INTO accounts VALUES ('A3', 2000);
COMMIT;

-- Transaction 1 re-executes:
SELECT * FROM accounts WHERE balance > 1000; -- Now sees the new row (phantom read)
```

---

## Oracle-Specific Transaction Quirks

- **Auto-Commit Behavior:**
	- By default, SQL*Plus and some tools auto-commit after each statement unless wrapped in a transaction block.
- **Implicit Commits:**
	- DDL statements (`CREATE`, `ALTER`, `DROP`, etc.) cause an implicit `COMMIT` before and after execution.
	- Some DCL statements (`GRANT`, `REVOKE`) also cause implicit commits.
- **Savepoints:**
	- You can roll back to a savepoint, but all changes after the savepoint are undone.
- **Locks:**
	- Uncommitted changes hold locks; committing or rolling back releases them.
- **Long Transactions:**
	- Holding transactions open for too long can cause lock contention and resource issues.

**Tip:** Always be aware of implicit commits and auto-commit settings in your SQL environment to avoid accidental data loss or partial updates.