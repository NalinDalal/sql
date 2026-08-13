# PL/SQL Security: Locking, Deadlock, and Exception Handling

> **Dialect note:** Oracle syntax throughout. Key differences from Postgres and MySQL are noted in gotchas.

---

### Concurrency Control and Locking

#### Explain It

Oracle tables are global resources accessed by multiple users simultaneously. **Concurrency control** ensures data integrity when many users read and write the same data at the same time.

Oracle uses **locks** — internal mechanisms that regulate concurrent access. Two categories:

- **Implicit locking** — Oracle applies locks automatically based on the operation type. No user action required.
- **Explicit locking** — The user manually acquires locks using `SELECT ... FOR UPDATE` or `LOCK TABLE` when they need tighter control over timing.

Lock modes:

| Lock type   | Purpose                    | Concurrent readers | Concurrent writers |
| ----------- | -------------------------- | ------------------ | ------------------ |
| Shared      | Read operations (`SELECT`) | Multiple allowed   | None               |
| Exclusive   | Write operations (`INSERT`, `UPDATE`, `DELETE`) | None (blocked) | None (blocked) |

Oracle does **not** provide field-level (column-level) locks. Locks are at the row or table level only.

#### Prove It

```sql
-- Implicit exclusive lock: UPDATE acquires a row lock automatically
UPDATE acct_mstr SET curb = 5000 WHERE acct_no = 'SB1';
-- Other sessions trying to UPDATE or SELECT FOR UPDATE this row will wait

-- Explicit row-level lock: SELECT ... FOR UPDATE
SELECT * FROM acct_mstr WHERE acct_no = 'SB1' FOR UPDATE NOWAIT;
-- NOWAIT returns ORA-00054 immediately if the row is already locked;
-- without NOWAIT, the session waits until the lock is released

-- Explicit table-level lock
LOCK TABLE acct_mstr IN EXCLUSIVE MODE NOWAIT;
-- Blocks all other access until COMMIT or ROLLBACK

-- Check for locked objects (DBA view)
SELECT session_id, lock_type, mode_held, mode_requested
  FROM v$locked_object;
```

#### Gotchas / Edge Cases

- `SELECT ... FOR UPDATE` **locks only the rows returned by the query** — adding `WHERE` conditions narrows the lock scope; omitting `WHERE` locks the whole table
- `NOWAIT` returns `ORA-00054: resource busy and acquire with NOWAIT specified` immediately; without it the session **blocks silently** until the lock is released or the session is killed
- Oracle's **exclusive lock on a row is released only at COMMIT or ROLLBACK** — not at the end of the statement. A long transaction holds row locks for its entire duration
- Postgres and MySQL also use row-level locks by default, but both allow **advisory locks** and explicit `GET_LOCK()` (MySQL) / `pg_advisory_lock()` (Postgres) that Oracle does not have
- Oracle has **no dirty reads** — even without explicit `FOR UPDATE`, a reader never sees uncommitted changes from another session (Oracle's default isolation is Read Committed)

---

### Lock Levels

#### Explain It

Oracle operates at two lock levels:

- **Row-level lock** — locks individual rows matching a `WHERE` clause. Most granular. Other sessions can modify rows that do not match the predicate.
- **Table-level lock** — locks the entire table. Used when a transaction touches many rows or needs to prevent DDL changes mid-transaction.

Oracle does **not** have page-level locks (some other databases do). Lock escalation from row to table is not automatic — the application must issue `LOCK TABLE` explicitly.

#### Prove It

```sql
-- Row-level: two sessions can update different rows concurrently
-- Session A:
UPDATE acct_mstr SET curb = 1000 WHERE acct_no = 'SB1';
-- Session B (concurrent, different row — succeeds):
UPDATE acct_mstr SET curb = 2000 WHERE acct_no = 'CA2';
-- Both hold exclusive row locks until their respective COMMITs

-- Table-level: blocks all DML on the table
-- Session A:
LOCK TABLE acct_mstr IN SHARE MODE;
-- Session B (concurrent UPDATE — waits until Session A COMMITs):
UPDATE acct_mstr SET curb = 0 WHERE acct_no = 'SB1';
```

#### Gotchas / Edge Cases

- A row-level lock **does not block SELECT** — other sessions can still read the locked row (they just cannot lock or modify it)
- `LOCK TABLE` in `SHARE MODE` allows **concurrent reads** but blocks writes — `EXCLUSIVE MODE` blocks both reads and writes from other sessions
- Oracle does **not escalate** row locks to table locks automatically (unlike SQL Server's lock escalation) — if you need a table lock, request it explicitly
- After `LOCK TABLE`, **all existing row locks in the session are released** — use with care in long transactions

---

### Deadlock

#### Explain It

A **deadlock** occurs when two or more transactions hold locks that each other needs, forming a cycle. Neither can proceed.

Oracle **detects and resolves deadlocks automatically**. It chooses a "victim" transaction and aborts it with `ORA-00060: deadlock detected while waiting for resource`. The other transaction proceeds. The victim can then retry.

This is different from a **lock wait** (one session waits for another), which is not a deadlock — Oracle does not time out lock waits by default.

#### Prove It

```sql
-- Session A
UPDATE acct_mstr SET curb = curb - 500 WHERE acct_no = 'SB1';
UPDATE acct_mstr SET curb = curb + 500 WHERE acct_no = 'CA2';
-- Session A holds row lock on SB1, waiting for row lock on CA2

-- Session B (concurrent)
UPDATE acct_mstr SET curb = curb - 500 WHERE acct_no = 'CA2';
UPDATE acct_mstr SET curb = curb + 500 WHERE acct_no = 'SB1';
-- Session B holds row lock on CA2, waiting for row lock on SB1
-- Oracle detects the cycle → raises ORA-00060 in one session

-- Handle deadlock in application code
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE = -60 THEN
      DBMS_OUTPUT.PUT_LINE('Deadlock detected — retry transaction');
    END IF;
END;
```

#### Gotchas / Edge Cases

- Oracle **always resolves deadlocks automatically** — the application never needs to intervene; it just retries the aborted transaction
- Postgres also detects deadlocks automatically (raises `SQLSTATE 40P01`); MySQL's InnoDB does too but rolls back the **smallest** transaction
- A **long wait without a deadlock** (single waiter, single holder) will block indefinitely in Oracle unless `NOWAIT` or a timeout (`ALTER SYSTEM SET dml_lock_timeout = 5`) is set — this is not a deadlock
- `ORA-00060` messages are written to the **alert log and trace file** — check `USER_DUMP_DEST` for the deadlock graph after an incident

---

### Exception Handling in PL/SQL

#### Explain It

Exceptions are runtime errors that interrupt normal execution. PL/SQL handles them in the `EXCEPTION` section of a block. Oracle raises exceptions for constraint violations (`ORA-02290`), logic errors (`NO_DATA_FOUND`), deadlocks (`ORA-00060`), and many other conditions.

Two categories:
- **Predefined (named) exceptions** — Oracle assigns a name to common errors (`NO_DATA_FOUND`, `TOO_MANY_ROWS`, `DUP_VAL_ON_INDEX`, etc.)
- **User-defined exceptions** — declared in the `DECLARE` section, raised with `RAISE` or `RAISE_APPLICATION_ERROR(-20000..-20999, 'msg')`

`PRAGMA EXCEPTION_INIT(name, error_code)` binds a numeric Oracle error code to a named exception so it can be caught by name.

#### Prove It

```sql
-- Named predefined exception
DECLARE
  v_bal NUMBER;
BEGIN
  SELECT curb INTO v_bal FROM acct_mstr WHERE acct_no = 'SB1';
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('Account not found');
END;
/

-- User-defined exception with PRAGMA EXCEPTION_INIT
DECLARE
  e_low_balance EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_low_balance, -20001);
BEGIN
  UPDATE acct_mstr SET curb = 500 WHERE acct_no = 'SB1';
  IF SQL%ROWCOUNT = 0 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Account not found for update');
  END IF;
EXCEPTION
  WHEN e_low_balance THEN
    DBMS_OUTPUT.PUT_LINE(SQLERRM);
END;
/

-- Business rule exception with RAISE
DECLARE
  e_insufficient_funds EXCEPTION;
BEGIN
  IF :new.curb < 0 THEN
    RAISE e_insufficient_funds;
  END IF;
EXCEPTION
  WHEN e_insufficient_funds THEN
    DBMS_OUTPUT.PUT_LINE('Withdrawal would result in negative balance');
END;
/
```

#### Gotchas / Edge Cases

- `NO_DATA_FOUND` raised by `SELECT INTO` with **zero rows** — not an error in the Oracle sense, but it does abort the block unless caught; use an aggregate (`NVL(MAX(col), default)`) to avoid it when zero rows is acceptable
- `TOO_MANY_ROWS` raised by `SELECT INTO` with **more than one row** — always add a unique-key `WHERE` clause or use `ROWNUM = 1` to prevent this
- `OTHERS` **must be the last** handler — any handler after it is unreachable and raises a compile error
- `SQLCODE` and `SQLERRM` are **only valid inside an EXCEPTION handler** — referencing them in the executable section raises `PLS-00201: identifier must be declared`
- `SQL%ROWCOUNT` is available after DML and gives the exact rows affected — useful for confirming an `UPDATE` or `DELETE` hit the expected row count

---

### Summary Table

| Concept              | Oracle Syntax / Behaviour                        | Postgres / MySQL difference                  |
| -------------------- | ------------------------------------------------ | -------------------------------------------- |
| Shared lock           | `SELECT ...` (Read Committed isolates readers)    | Same by default in both                      |
| Exclusive lock        | Implicit on DML; explicit via `FOR UPDATE`        | Same                                         |
| Table lock            | `LOCK TABLE t IN EXCLUSIVE MODE [NOWAIT]`        | `LOCK TABLE t` (Postgres); `LOCK TABLES t` (MySQL) |
| Deadlock              | Auto-detected; `ORA-00060`; app retries           | Same behaviour; different error codes        |
| Row lock release      | At `COMMIT` / `ROLLBACK` only                    | Same in InnoDB/Postgres                      |
| Named exception       | `NO_DATA_FOUND`, `DUP_VAL_ON_INDEX`, ...          | Similar names, different codes               |
| User-defined error    | `RAISE_APPLICATION_ERROR(-20000..-20999, msg)`   | Postgres: `RAISE EXCEPTION 'msg'` using SQLSTATE |
| `PRAGMA EXCEPTION_INIT` | Binds error code to named exception            | Postgres: no direct equivalent; use SQLSTATE |
