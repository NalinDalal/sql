# SQL Commands

> **Dialect note:** Oracle SQL (VARCHAR2, DUAL, FETCH FIRST). PostgreSQL/MySQL differences are flagged inline where they matter in interviews. (Chapter 7 of the course PDF.)

### Data Types
#### Explain It
Oracle's basic string types are `CHAR(size)`, a fixed-length string that pads short values with spaces, and `VARCHAR2(size)`, a variable-length string that stores only what you give it. The classic textbook claim that "CHAR is up to 50% faster than VARCHAR2" is **unverified folklore — do not repeat a specific number in an interview**; the real reason to prefer CHAR is data that is naturally fixed-width (codes, flags, hashes). `NUMBER` covers integers and decimals, and `DATE` stores date + time.

#### Prove It
```sql
CREATE TABLE DEMO_TYPES (
  FIXED   CHAR(10),      -- 'AB' is stored as 'AB        ' (padded)
  FLEX    VARCHAR2(10),  -- 'AB' is stored as 'AB'
  QTY     NUMBER(6,2),
  OCCURRED DATE
);
```

#### Gotchas / Edge Cases
- Oracle compares `CHAR` values with blank-padded comparison, so `CHAR(10)` holding `'AB'` equals `'AB'` — but `VARCHAR2` does not pad, which causes surprising inequality when mixing the two types.
- `VARCHAR2(4000)` is the classic max in a table column (older limit); `VARCHAR( n )` is an alias that will become `VARCHAR2` in the future — never use it, use `VARCHAR2` directly.
- MySQL's `VARCHAR` is fine, but Oracle text fields are `VARCHAR2` — a classic interview trap if you mix dialects.

---

### CREATE TABLE and the "Schema = Username" Rule
#### Explain It
Oracle has no separate "schema" object: a schema is the collection of objects owned by a database user, and the schema name *is* the username. So the course's `CREATE SCHEMA "DBA_BANKSYS"` + `SET search_path` idea (which is PostgreSQL syntax) maps to Oracle as: create a user named DBA_BANKSYS, then create the tables while connected as that user (or by qualifying the object name with the username). Table names follow rules: max 30 chars, start with a letter, alphanumeric only, no reserved words, no special characters.

#### Prove It
```sql
-- run as a DBA (e.g. SYSTEM):
CREATE USER DBA_BANKSYS IDENTIFIED BY bankpass;
GRANT CONNECT, RESOURCE, UNLIMITED TABLESPACE TO DBA_BANKSYS;
```
```sql
-- connect as DBA_BANKSYS and create the table in your own schema:
CREATE TABLE BRANCH_MSTR (
  BRANCH_NO VARCHAR2(20),
  NAME      VARCHAR2(25)
);
```

#### Gotchas / Edge Cases
- A brand-new Oracle user has **zero quota** on the USERS tablespace: the first INSERT fails with `ORA-01950: insufficient quota` until you grant `UNLIMITED TABLESPACE` (or a quota).
- The old PostgreSQL shortcut `CREATE SCHEMA x; SET search_path TO x;` does **not** exist in Oracle — qualify with `owner.table` (`DBA_BANKSYS.BRANCH_MSTR`) instead.
- In an interview, "schema = username" is a crisp Oracle fact; PostgreSQL separates logins (roles) from database objects (schemas).

---

### INSERT
#### Explain It
`INSERT` adds a new empty row to a table and loads the values you pass into the listed columns. The column list and the `VALUES` list must line up positionally; columns you omit get `NULL` (or their `DEFAULT`). You can also insert many rows at once by selecting them from another table.

#### Prove It
```sql
INSERT INTO BRANCH_MSTR (BRANCH_NO, NAME) VALUES ('B1', 'Vile Parle (HO)');
INSERT INTO BRANCH_MSTR (BRANCH_NO, NAME) VALUES ('B3', 'Churchgate');
INSERT INTO BRANCH_MSTR (BRANCH_NO, NAME) VALUES ('B4', 'Sion');
```
Oracle prints something like `1 row created.` after each statement (psql prints `INSERT 0 1` — output format is client-specific).

#### Gotchas / Edge Cases
- Column/value count mismatch is a runtime error (`ORA-00947: not enough values`) — the classic "forgot one column" trap.
- Strings are single-quoted; a literal single quote inside is doubled (`'O''Brien'`), not escaped with a backslash like MySQL.
- `INSERT` inside a transaction can be rolled back; `INSERT` performed by DDL (CTAS) cannot (see TRUNCATE/DELETE concept).

---

### SELECT — the Query Language
#### Explain It
`SELECT` is the heart of SQL: it retrieves data and builds a temporary result set. Execution order matters — the `FROM` clause is processed first (which tables feed the query), then `WHERE` filters rows, and only then is the `SELECT` list projected. `SELECT *` returns every column; listing columns returns only those.

#### Prove It
```sql
-- all columns, all rows
SELECT * FROM BRANCH_MSTR;

-- selected columns, all rows
SELECT BRANCH_NO FROM BRANCH_MSTR;

-- all columns, selected rows
SELECT * FROM BRANCH_MSTR WHERE NAME = 'Churchgate';

-- selected columns AND selected rows
SELECT ORDER_ID, ITEM FROM ORDERS WHERE AMOUNT > 300;
```

#### Gotchas / Edge Cases
- `WHERE <col> = NULL` matches **nothing** — NULL is not a value, so compare with `IS NULL` / `IS NOT NULL` (see first interview question in `10-interview-questions.md`).
- In Oracle you may write `SELECT 2 FROM DUAL;` — `DUAL` is Oracle's dummy one-row table for function demos; Postgres/MySQL use `SELECT 2;` without FROM.
- A `SELECT` never modifies data — if the interviewer asks "does SELECT lock rows?", Oracle's plain SELECT is lock-free (except `FOR UPDATE`, see `security.md`).

---

### Returning Only the First N Rows
#### Explain It
To restrict the result to a limited number of rows, the modern Oracle keyword is `FETCH FIRST n ROWS ONLY` (Oracle 12c+). Older Oracle used the `ROWNUM` trick (see `advance-feat.md`); SQL Server uses `SELECT TOP 3`, PostgreSQL and MySQL use `LIMIT 3` — naming all three in an interview scores points.

#### Prove It
```sql
SELECT ORDER_ID, ITEM FROM ORDERS
ORDER BY AMOUNT DESC
FETCH FIRST 2 ROWS ONLY;
```

#### Gotchas / Edge Cases
- Without an `ORDER BY`, "first 3" is arbitrary — the database is free to return any 3 rows.
- `FETCH FIRST` is ANSI SQL (works in Postgres too), but SQL Server's `TOP` and MySQL's `LIMIT` are not interchangeable — knowing the mapping is a classic interview question.
- The old `WHERE ROWNUM < 4` approach breaks when combined with `ORDER BY` (explained in `advance-feat.md`).

---

### DISTINCT — Removing Duplicate Rows
#### Explain It
`DISTINCT` removes rows that are exactly identical across every selected column; it can only be used with `SELECT`. Duplicates are considered on the full row, not per column, so `SELECT DISTINCT ITEM, AMOUNT` keeps rows that differ in *either* column.

#### Prove It
```sql
SELECT DISTINCT ITEM FROM ORDERS;
```

#### Gotchas / Edge Cases
- `DISTINCT` scans and compares whole rows — on big tables it can be slower than a plain `GROUP BY` on the same columns (both give the same answer here).
- `COUNT(DISTINCT col)` counts unique values while ignoring NULLs.
- In Oracle, `SELECT DISTINCT` on a column with NULLs keeps one NULL row (NULL = NULL for grouping purposes).

---

### ORDER BY — Sorting Results
#### Explain It
`ORDER BY` sorts the result set by one or more columns, ascending (default) or descending with `DESC`. It is the last clause to execute, which is why it can sort by a column alias defined in the `SELECT` list.

#### Prove It
```sql
SELECT * FROM CUSTOMERS ORDER BY FIRST_NAME;
SELECT * FROM CUSTOMERS ORDER BY FIRST_NAME DESC;
SELECT ORDER_ID, ITEM FROM ORDERS ORDER BY AMOUNT DESC FETCH FIRST 2 ROWS ONLY; -- 2 most recent/largest
```

#### Gotchas / Edge Cases
- Multiple sort keys are applied left to right; each key can have its own `ASC`/`DESC`.
- NULLs sort last in Oracle's ascending order (first in some other databases — e.g. MySQL puts NULLs first).
- An `ORDER BY` on a non-indexed column can force a full sort of the result — relevant when an index "hurts or helps" (see `advanced-sql.md`).

---

### Creating a Table From a Table (CTAS)
#### Explain It
`CREATE TABLE ... AS SELECT` (CTAS) both creates a new table and populates it with the query's result in one statement. The new table inherits the column *names* and *types* of the query output, but generally not constraints, defaults, or indexes.

#### Prove It
```sql
CREATE TABLE ORDERS2 AS SELECT ORDER_ID, ITEM, AMOUNT FROM ORDERS;
```

#### Gotchas / Edge Cases
- No constraints are copied (no PK, no NOT NULL) — a lighter "copy" than `CREATE TABLE ... LIKE` style available elsewhere.
- CTAS is DDL: in Oracle it implicitly commits the transaction and cannot be rolled back.
- `SELECT *` inside CTAS snapshots the current data — it does not stay in sync when the source table changes.

---

### Inserting Data From Another Table
#### Explain It
Instead of `VALUES`, an `INSERT` can take its rows from a `SELECT` — the query supplies one row set that is appended in a single statement. Column lists must still line up: number and types of columns in the query must match the insert target.

#### Prove It
```sql
INSERT INTO ORDERS2 (ORDER_ID, ITEM, AMOUNT)
SELECT ORDER_ID, ITEM, AMOUNT FROM ORDERS;
```

#### Gotchas / Edge Cases
- This is the standard "archive the old rows" pattern: move rows out of a hot table into a history table.
- If the SELECT returns more rows than the target allows (PK conflicts), the whole statement fails in Oracle — DML statements are atomic.

---

### DELETE
#### Explain It
`DELETE` removes rows (all of them, or a filtered set). The table structure and the space stay allocated. Because you cannot list two tables in the `FROM` clause of a DELETE, deleting rows based on another table's data uses a subquery — classically an `EXISTS` subquery.

#### Prove It
```sql
-- all rows
DELETE FROM ORDERS2;

-- rows matching a condition from ANOTHER table
DELETE FROM ADDR_DTLS WHERE EXISTS (SELECT FNAME FROM CUST_MSTR
              WHERE CUST_MSTR.CUST_NO = ADDR_DTLS.CODE_NO
                AND CUST_MSTR.FNAME = 'Ivan');
```

#### Gotchas / Edge Cases
- `DELETE` without `WHERE` empties the table — always double-check before running it in production.
- DELETE is DML and transactional: it can be rolled back in Oracle (TRUNCATE cannot — next concept).
- In Oracle a `DELETE` that hits a row referenced as a parent key fails with a referential-integrity error unless `ON DELETE CASCADE` is set (see `constraints.md`).

---

### UPDATE
#### Explain It
`UPDATE` changes values in existing rows. The `SET` clause lists which columns get which new values; with no `WHERE`, every row in the table is updated; with a `WHERE`, only matching rows.

#### Prove It
```sql
-- all rows
UPDATE BRANCH_MSTR SET NAME = UPPER(NAME);

-- only selected rows
UPDATE BRANCH_MSTR SET NAME = 'Head Office'
WHERE NAME = 'Vile Parle (HO)';
```

#### Gotchas / Edge Cases
- Missing `WHERE` = unintentional mass update; the classic interview trap "what happens if you run UPDATE without WHERE?" — all rows change (and are reversible only if not yet committed).
- Oracle does not allow updating a joined view directly in all cases (see view restrictions in `advanced-sql.md`).
- `UPDATE` holds row locks until COMMIT/ROLLBACK — a long transaction blocks other writers (see `security.md`).

---

### ALTER TABLE — Modifying Structure
#### Explain It
`ALTER TABLE` changes an existing table's structure. Oracle makes a temporary copy of the table, performs the alteration on the copy, and swaps it in. You can add columns, drop columns, and modify a column's type/size. The classic limitations: you cannot rename the table or a column with ALTER (use the dedicated `RENAME` statement), and you cannot shrink a column below existing data.

#### Prove It
```sql
ALTER TABLE BRANCH_MSTR ADD (CITY VARCHAR2(25));
ALTER TABLE BRANCH_MSTR DROP COLUMN CITY;
ALTER TABLE BRANCH_MSTR MODIFY (NAME VARCHAR2(30));
```

#### Gotchas / Edge Cases
- Oracle syntax is `ALTER TABLE t MODIFY (cols);` — PostgreSQL says `ALTER TABLE t ALTER COLUMN c TYPE ...`; MySQL says `MODIFY COLUMN`. Same intent, three spellings.
- Dropping a column that is part of the PRIMARY KEY is refused; drop the constraint first.
- `ALTER` is DDL → implicit commit in Oracle (see `transactions.md`).

---

### RENAME
#### Explain It
A standalone `RENAME old TO new` statement changes the name of a table (in PostgreSQL it's `ALTER TABLE old RENAME TO new`). Existing synonyms and views that reference it are invalidated until you update them.

#### Prove It
```sql
RENAME BRANCH_MSTR TO BRANCHES;
```

#### Gotchas / Edge Cases
- Oracle's `RENAME` also works on sequences and synonyms; for columns it's `ALTER TABLE t RENAME COLUMN a TO b;` (12c+).
- Renaming breaks stored procedures/views that reference the old name — they go invalid until recompiled (see `db-obj.md`).

---

### TRUNCATE vs DELETE
#### Explain It
`TRUNCATE TABLE` empties a table much faster than DELETE because it doesn't log/process rows one by one — Oracle drops and re-creates the table's storage. It does not return a count of deleted rows, and it is **transaction-safety differs by dialect: in Oracle TRUNCATE is DDL — it issues an implicit COMMIT and cannot be rolled back; in PostgreSQL TRUNCATE is transactional and can be rolled back.** Saying the Oracle version and then adding "but Postgres differs" is exactly what an interviewer wants to hear.

#### Prove It
```sql
TRUNCATE TABLE BRANCH_MSTR;   -- same result as DELETE without WHERE, but:
--   Oracle:  not rollback-able, table structure stays
--   Postgres: rollback-able
```

#### Gotchas / Edge Cases
- TRUNCATE cannot run on a table with active locks/transactions referencing it in some databases (`cannot truncate a table referenced in a foreign key` — child tables must be truncated first or the FK disabled).
- DELETE fires row-level triggers; TRUNCATE generally does not fire per-row triggers.
- "DELETE without WHERE vs TRUNCATE" is a top interview pair: transactional vs not (Oracle), fast vs slow, per-row triggers vs storage reset.

---

### DROP TABLE — Destroying a Table
#### Explain It
`DROP TABLE` removes the table definition and all its data permanently. It cannot be undone (unless recovery/backup), so it is the most destructive of the "structure" commands.

#### Prove It
```sql
DROP TABLE BRANCH_MSTR;
```

#### Gotchas / Edge Cases
- Oracle's `DROP TABLE` on a table referenced by foreign keys fails unless you add `CASCADE CONSTRAINTS`.
- `DROP TABLE ... PURGE` bypasses the recycle bin (Oracle) for immediate, unrecoverable removal.
- TRUNCATE keeps the structure; DROP removes the structure — that one-liner difference is a favorite interview short answer.

---

### Synonyms
#### Explain It
A synonym is an alternative name for a database object (table, view, sequence, procedure…) that hides the real owner and object name. `CREATE PUBLIC SYNONYM` makes it available to all users (who still need their own privileges on the underlying object); a plain synonym is private to its owner.

#### Prove It
```sql
CREATE SYNONYM ORD_ALIAS FOR ORDERS;          -- private
SELECT COUNT(*) FROM ORD_ALIAS;               -- same as SELECT COUNT(*) FROM ORDERS

CREATE PUBLIC SYNONYM ORD_PUB FOR ORDERS;     -- all users can reference ORD_PUB
DROP PUBLIC SYNONYM ORD_PUB;

DROP SYNONYM ORD_ALIAS;
```

#### Gotchas / Edge Cases
- Creating a public synonym needs the `CREATE PUBLIC SYNONYM` privilege, and dropping it needs `DROP PUBLIC SYNONYM` — a privilege mismatch is a common ORA-01031 surprise.
- A public synonym is a CDB-level object in newer Oracle (PDB restrictions apply) — flag this only if the interviewer goes deep.
- Synonyms do not grant access: `GRANT` on the underlying object is still required (see `permissions.md`).

---

### DESCRIBE — Displaying Table Structure
#### Explain It
`DESCRIBE table` (or `DESC table`) is the SQL\*Plus/SQLcl client command that prints a table's columns, types, and nullability. It's the fastest way to inspect a schema you don't know — the interview answer to "how do you see a table's structure?" is DESCRIBE in Oracle; psql uses `\d table` and MySQL `SHOW COLUMNS FROM table`.

#### Prove It
```sql
DESCRIBE BRANCH_MSTR;
```

#### Gotchas / Edge Cases
- DESCRIBE is a *client* command, not standard SQL — it won't run through a JDBC driver or other tools without the sqlplus shell.
- DESCRIBE shows NOT NULL but not PRIMARY KEY — for constraints, query `USER_CONSTRAINTS`/`USER_CONS_COLUMNS` (or `\d table` details in psql).