# Transactions

> **Dialect note:** Oracle — transactions are implicit (no `BEGIN` needed); PL/SQL `BEGIN ... END;` is a code block, not a transaction marker.

### What is a Transaction
#### Explain It
A transaction groups logically related SQL statements into a single unit of work. Oracle treats changes in two steps: the changes are made, then they are made permanent with `COMMIT` or undone with `ROLLBACK`. A transaction starts with the first executable statement after a connection/commit, and ends at the next `COMMIT`/`ROLLBACK`. All locks taken during the transaction are released only at that point.

#### Prove It
```sql
UPDATE ACCT_MSTR SET CURBAL = CURBAL - 1000 WHERE ACCT_NO = 'SB1';
SELECT CURBAL FROM ACCT_MSTR WHERE ACCT_NO = 'SB1';   -- 9000 (your own session sees it)
ROLLBACK;                                              -- undo it
SELECT CURBAL FROM ACCT_MSTR WHERE ACCT_NO = 'SB1';   -- 10000 again
```

#### Gotchas / Edge Cases
- **Oracle starts a transaction implicitly** — there is no `BEGIN TRANSACTION`; the first DML statement opens one. PostgreSQL/MySQL require an explicit `BEGIN;` / `START TRANSACTION;` — knowing which dialect does what is a favorite interview point.
- Uncommitted changes are visible to *you* but to no one else; other sessions keep seeing the committed version (read consistency).
- If the session disconnects without COMMIT, the transaction is rolled back automatically (and locks released).

---

### COMMIT and ROLLBACK
#### Explain It
`COMMIT` makes every change in the current transaction permanent and releases the transaction's locks. `ROLLBACK` undoes every change since the transaction started, restoring the data to its last committed state, and also releases locks. There is no "undo after commit" — anything committed survives even a crash (that is the Durability leg of ACID).

#### Prove It
```sql
UPDATE ACCT_MSTR SET CURBAL = CURBAL - 1000 WHERE ACCT_NO = 'SB1';
COMMIT;          -- now permanent — a later ROLLBACK cannot undo it
SAVEPOINT s1;
UPDATE ACCT_MSTR SET CURBAL = 0 WHERE ACCT_NO = 'SB1';
ROLLBACK;        -- undoes only what happened AFTER the COMMIT
```

#### Gotchas / Edge Cases
- The order matters in interviews: "can a ROLLBACK undo a committed change?" — no.
- Both commands release row/table locks; a lingering uncommitted UPDATE is the classic "why is my query hanging" cause (see `security.md`).
- DDL (CREATE/ALTER/DROP/TRUNCATE) implicitly commits *before and after* itself — the reason you can't roll back a CREATE TABLE.

---

### SAVEPOINT — Partial Rollback
#### Explain It
A savepoint marks a point inside a transaction. `ROLLBACK TO SAVEPOINT name` undoes only the statements issued after that marker, keeping the earlier ones. This lets you retry a step without losing the whole unit of work. Rolling back to a savepoint does **not** end the transaction and does **not** release locks.

#### Prove It
```sql
UPDATE ACCT_MSTR SET CURBAL = CURBAL - 1000 WHERE ACCT_NO = 'SB1';   -- withdrawal: 10000 -> 9000
SAVEPOINT after_withdrawal;
UPDATE ACCT_MSTR SET CURBAL = CURBAL + 1000 WHERE ACCT_NO = 'CA2';   -- deposit:   2500 -> 3500
ROLLBACK TO SAVEPOINT after_withdrawal;                               -- deposit undone, withdrawal kept
COMMIT;                                                               -- SB1 stays 9000, CA2 back to 2500
```

#### Gotchas / Edge Cases
- After `ROLLBACK TO SAVEPOINT`, the savepoint still exists (you can reuse or re-create it); the markers created after it are gone.
- Locks are NOT released by a partial rollback — only the full `COMMIT`/`ROLLBACK` frees them.
- Savepoints work only inside a transaction; a `COMMIT` destroys all savepoints.

---

### ACID Properties
#### Explain It
The four guarantees a transaction engine must provide: **Atomicity** — all statements in the transaction succeed or none of them do; **Consistency** — the transaction moves the database from one valid state to another, preserving every constraint; **Isolation** — concurrent transactions do not see each other's intermediate results; **Durability** — once committed, changes survive crashes and power loss. Interviews usually ask for the expansion plus a one-line example of each from the banking domain.

#### Prove It
```sql
BEGIN -- (Postgres/MySQL need the explicit marker; Oracle does not)
  UPDATE ACCT_MSTR SET CURBAL = CURBAL - 1000 WHERE ACCT_NO = 'SB1';  -- atomic with:
  UPDATE ACCT_MSTR SET CURBAL = CURBAL + 1000 WHERE ACCT_NO = 'CA2';  -- this one
  COMMIT;  -- durable only from this moment
EXCEPTION    -- Or: PL/SQL error handling rolls back the pair
  WHEN OTHERS THEN ROLLBACK;
END;
```

#### Gotchas / Edge Cases
- Atomicity is about *statements within one transaction*, not about a single UPDATE touching many rows (that is one statement — also atomic in Oracle).
- The "consistency" C is what CHECK/NOT NULL/FK constraints protect (see `constraints.md`).
- Durability mechanism differs: Oracle uses redo logs; Postgres uses WAL — same idea, different names.

---

### Isolation Levels in Oracle vs Postgres vs MySQL
#### Explain It
The isolation level controls how much of other transactions' uncommitted/committed work a transaction sees. **Oracle supports only READ COMMITTED (default) and SERIALIZABLE.** The other two standard levels exist in other engines — remember the full ANSI ladder and which level each vendor defaults to.

| Level | Oracle | PostgreSQL | MySQL |
|---|---|---|---|
| READ UNCOMMITTED | not supported | supported (behaves like READ COMMITTED) | supported (default is REPEATABLE READ) |
| READ COMMITTED | **default** | **default** | supported |
| REPEATABLE READ | not supported | supported | **default** |
| SERIALIZABLE | supported | supported | supported |

#### Prove It
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;   -- Oracle: one of the only two legal values
SELECT COUNT(*) FROM ACCT_MSTR;
COMMIT;
```

#### Gotchas / Edge Cases
- Oracle's `SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED` is a **runtime error** — flagging this surprises interviewers who assume all engines support all four levels.
- Oracle's "read consistency" means a READ COMMITTED query sees a consistent snapshot from when the query started — no dirty reads, ever.
- Postgres accepts READ UNCOMMITTED syntactically but runs it as READ COMMITTED (no dirty reads there either); MySQL's InnoDB actually does negotiate its own REPEATABLE-READ-style snapshotting.
- Raising the level trades concurrency for correctness — the classic "why not always SERIALIZABLE" answer is lock contention/staleness.

---

### Isolation Anomalies: Dirty, Non-Repeatable, Phantom Reads
#### Explain It
Lower isolation levels allow anomalies: **Dirty Read** — seeing another transaction's *uncommitted* change (impossible in Oracle, only possible in READ UNCOMMITTED levels); **Non-Repeatable Read** — the same row read twice inside one transaction returns different values because another transaction committed a change in between (possible in READ COMMITTED, avoided in REPEATABLE READ/SERIALIZABLE); **Phantom Read** — re-running a range query returns a *different set of rows* because another transaction inserted/removed matching rows in between.

#### Prove It
```sql
-- session-level illustration (concept): the classic money-transfer pair is safe even at
-- READ COMMITTED; the anomalies show up when a query is repeated inside one transaction
SELECT SUM(CURBAL) FROM ACCT_MSTR;   -- T1: 100 .. another session commits a new account ..
SELECT SUM(CURBAL) FROM ACCT_MSTR;   -- T1: 150  -> phantom read (in READ COMMITTED)
```
(To actually see the difference you need two concurrent sessions; the anomaly definitions above are what the interviewer checks.)

#### Gotchas / Edge Cases
- Oracle's READ COMMITTED is statement-level snapshot: repeat *queries* can still see new data (phantom) — Oracle's serializable cannot see it for the transaction's lifetime.
- ORDER of danger: dirty read is hardest to defend, phantom read easiest — interviewers often ask you to order them.
- These same four words (dirty/non-repeatable/phantom) are defined in `transactions.md` and rehearsed in `10-interview-questions.md` — reuse the same definitions for consistency.

---

### Oracle Transaction Quirks (Implicit Commits)
#### Explain It
Several Oracle behaviors surprise newcomers: DDL statements (`CREATE`, `ALTER`, `DROP`, `TRUNCATE`) cause an **implicit COMMIT** before and after themselves; some DCL statements (`GRANT`, `REVOKE`) do too. SQL\*Plus/other tools may auto-commit statements unless you wrap them in a logical unit, and long-running uncommitted transactions hold locks and bloat undo segments.

#### Prove It
```sql
UPDATE ACCT_MSTR SET CURBAL = 0;      -- would roll back fine...
CREATE TABLE TMP_PROBE (id NUMBER);   -- ...but this DDL implicitly COMMITS first
ROLLBACK;                             -- the update is now permanent. Oops.
DROP TABLE TMP_PROBE;
```

#### Gotchas / Edge Cases
- "Heart of the trap question": UPDATE, then CREATE, then ROLLBACK — the update survives.
- `GRANT`/`REVOKE` also commit implicitly (DCL), so sloppy scripts half-commit.
- Long transactions = lock contention (`security.md`) + undo pressure; commit in batches in batch jobs.

---

### Transaction Error Handling (PL/SQL)
#### Explain It
In PL/SQL, the `EXCEPTION` block catches errors raised while the transaction runs. The standard pattern: try the statements, `COMMIT` on success, and on any error `ROLLBACK` and log the failure — guaranteeing atomicity even when the code, not the database, detects the problem.

#### Prove It
```sql
SET SERVEROUTPUT ON
BEGIN
  UPDATE ACCT_MSTR SET CURBAL = CURBAL - 1000 WHERE ACCT_NO = 'SB1';
  UPDATE ACCT_MSTR SET CURBAL = CURBAL + 1000 WHERE ACCT_NO = 'CA2';
  COMMIT;
  DBMS_OUTPUT.PUT_LINE('Transaction committed.');
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    DBMS_OUTPUT.PUT_LINE('Error occurred. Transaction rolled back.');
END;
/
```

#### Gotchas / Edge Cases
- `WHEN OTHERS` catches everything — but swallowing errors silently is an anti-pattern; log or re-raise.
- `ROLLBACK` inside the handler releases locks only when it completes; keep the handler short.
- This is the PL/SQL version of atomicity; the pure-SQL version needs no handler because a failed statement rolls itself back (statement-level atomicity).