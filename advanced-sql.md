# Advanced SQL Notes

> **Dialect note:** Oracle (ROWID, ROWNUM, bitmap/function-based indexes, sequences with NEXTVAL/CURRVAL, snapshots/MVs).

### Indexes — Speeding Up Retrieval
#### Explain It
An index is an ordered structure built from a column's (or columns') values that lets the database find rows without scanning the whole table. Oracle keeps an address field per entry — the **ROWID** — pointing at the exact storage location of each record, so a lookup becomes "search the index, jump to the row". Indexes shine for `WHERE` filters, `ORDER BY`, and join keys; they cost space and slow down INSERT/UPDATE/DELETE because every write must also maintain the index.

#### Prove It
```sql
SELECT ROWID, BRANCH_NO FROM BRANCH_MSTR WHERE ROWNUM < 2;  -- ROWID: physical address pointer

CREATE INDEX idx_branch_name ON BRANCH_MSTR(NAME);  -- an ordered copy of NAME + address
SELECT * FROM BRANCH_MSTR WHERE NAME = 'Dadar';     -- now resolvable via the index
DROP INDEX idx_branch_name;
```

#### Gotchas / Edge Cases
- ROWID is not stored in the table itself — it is Oracle's internal locator and can change after reorganization (don't persist it).
- An index only helps when the engine chooses it; on tiny tables Oracle plans a full scan anyway.
- `WHERE UPPER(col) = 'X'` cannot use an index on `col` — the expression differs from the index key unless a function-based index exists (below). A super-common interview trap.

---

### B-tree Index — The Default
#### Explain It
The default index type in Oracle (and nearly every relational engine) is the **B-tree**: a balanced tree where leaf blocks hold the actual key values plus ROWIDs, ordered so the engine can binary-search instead of linearly scanning. "Balanced" means all leaves are the same distance from the root, so lookup cost stays ~logarithmic even as data grows. B-trees are great for equality and range queries (`=`, `>`, `BETWEEN`, `LIKE 'abc%'`) and for keys with many distinct values.

#### Prove It
```sql
CREATE INDEX idx_orders_amount ON ORDERS(AMOUNT);   -- defaults to B-tree
SELECT * FROM ORDERS WHERE AMOUNT BETWEEN 500 AND 2000;   -- range seek: index-friendly
DROP INDEX idx_orders_amount;
```

#### Gotchas / Edge Cases
- B-tree entry cost is balanced: leaf ≤ 50% full after splits; heavy deletions cause index fragmentation.
- `LIKE '%abc'` (leading wildcard) defeats a B-tree — it cannot range-seek a suffix; full scan or full-text index needed.
- "Why is the default index a B-tree and not a hash?" is a stock interview question: B-tree supports ranges and ordering, hash supports only equality.

---

### Index Types
#### Explain It
Beyond the default B-tree, Oracle offers: **unique** (enforces uniqueness, backs PK/UNIQUE), **composite** (one index over several columns; the leading column drives the search), **reverse key** (stores values reversed to spread inserts across blocks), **bitmap** (stores bitmaps per distinct value — superb for low-cardinality columns in OLAP, terrible for write-heavy tables), **function-based** (indexes an expression like `UPPER(FNAME)`, letting `WHERE UPPER(...)` use it), and **key-compressed** (shares common prefix entries to save space).

#### Prove It
```sql
CREATE UNIQUE INDEX idx_emp_names   ON EMP_MSTR(FNAME, LNAME);
CREATE BITMAP INDEX idx_cust_occ    ON CUST_MSTR(OCCUP);          -- 2-3 distinct values: bitmap territory
CREATE INDEX     idx_emp_upper      ON EMP_MSTR(UPPER(FNAME));    -- function-based
CREATE INDEX     idx_emp_compress   ON EMP_MSTR(DEPT, DESIG) COMPRESS 1;
CREATE INDEX     idx_emp_branch     ON EMP_MSTR(BRANCH_NO, DEPT); -- composite
DROP INDEX idx_emp_names;
DROP INDEX idx_cust_occ;
DROP INDEX idx_emp_upper;
DROP INDEX idx_emp_compress;
DROP INDEX idx_emp_branch;
```

#### Gotchas / Edge Cases
- A composite index serves queries that filter on its *leading* column(s) — filtering only on the second column ignores the index.
- Bitmap indexes on frequently-updated columns cause lock contention (each update touches many bitmap segments) — the "bitmap in OLTP is a mistake" line.
- `REVERSE` indexes break range scans along with the block-contention they prevent — use them only for equality lookups.
- Oracle automatically indexes UNIQUE/PK; manually adding another unique index on the same column list is an error (`ORA-01408`).

---

### When an Index HURTS (Worked Example)
#### Explain It
The textbook says "too many indexes slow inserts" — here is the mechanics. Every INSERT/UPDATE/DELETE must maintain *every* index on the table, so each additional index multiplies write cost; and an index on a low-cardinality column (few distinct values, e.g. `STATUS` with OPEN/CLOSED) is near-useless for reads because "30% of rows match" means the optimizer picks a full table scan over random index reads — you pay the write overhead and get no read benefit.

#### Prove It
```sql
-- a low-cardinality column: only 2 distinct values across many rows
CREATE TABLE ORDERS_STATUS AS SELECT * FROM ORDERS;
UPDATE ORDERS_STATUS SET STATUS = 'OPEN';
UPDATE ORDERS_STATUS SET STATUS = 'CLOSED' WHERE ORDER_ID <= 3;

CREATE INDEX idx_status ON ORDERS_STATUS(STATUS);   -- 2 distinct values: selectivity ~50%
-- with real-sized tables the planner ignores this index and full-scans:
--   SELECT COUNT(*) FROM ORDERS_STATUS WHERE STATUS = 'OPEN';
DROP TABLE ORDERS_STATUS;
```
```sql
-- write-heavy table: an extra index means extra work on EVERY insert/update
CREATE INDEX idx_cust_occup ON CUST_MSTR(OCCUP);
INSERT INTO CUST_MSTR VALUES ('C9','Test','T','Row', DATE '2000-01-01','Service',NULL,NULL,'N','N');
-- the insert above must also add OCCUP to idx_cust_occup
DROP INDEX idx_cust_occup;
```

#### Gotchas / Edge Cases
- Rule of thumb to quote: index when the column is used in WHERE/join/ORDER BY *and* is selective (few rows per value); don't index low-cardinality "flag/status" columns in OLTP.
- Each index adds memory and disk; big scans read indexes *plus* table blocks — worse than a clean full scan.
- Workload balance matters: analytics-heavy tables can afford many indexes; transaction tables want few.

---

### VIEWS — Virtual Tables
#### Explain It
A view is a named SELECT statement stored as a pseudo-table. It reduces redundancy (one definition, many uses), enforces security (expose selected columns only), and lets you hide complex joins behind a simple name. Views can be read-only or **updateable**: an updateable view must be based on one base table, include all NOT NULL columns, and avoid DISTINCT/GROUP BY/HAVING/aggregates/subqueries/set operators.

#### Prove It
```sql
CREATE VIEW vw_branch AS
SELECT BRANCH_NO, NAME FROM BRANCH_MSTR;

SELECT * FROM vw_branch;   -- looks like a table, always reflects current data

-- updateable: single table, all NOT NULL columns present
CREATE VIEW vw_cust AS SELECT CUST_NO, FNAME, MNAME FROM CUST_MSTR;
UPDATE vw_cust SET FNAME = 'IVAN UPD' WHERE CUST_NO = 'C3';
SELECT FNAME FROM CUST_MSTR WHERE CUST_NO = 'C3';   -- 'IVAN UPD': the view wrote through
DROP VIEW vw_branch;
DROP VIEW vw_cust;
```

#### Gotchas / Edge Cases
- A view stores **no data** — the classic interview contrast is "view = stored query, table = stored rows".
- Joins make views non-updateable unless one table is *key-preserved*; Oracle raises ORA-01779/01752/01732 for such attempts.
- `CREATE OR REPLACE VIEW` updates the definition without dropping it first.

---

### Master/Detail Views
#### Explain It
A view over a master/detail join (e.g. branch ↔ address) is *key-preserved* when each row of the detail maps to exactly one master row through the primary key. DML through such views is allowed only on the key-preserved (detail) table; inserting into the master side or updating non-preserved columns raises the classic errors.

#### Prove It
```sql
CREATE VIEW vw_branch_addr AS
SELECT B.BRANCH_NO, B.NAME, A.CITY
FROM BRANCH_MSTR B, ADDR_DTLS A
WHERE A.CODE_NO = B.BRANCH_NO;

SELECT * FROM vw_branch_addr;   -- branch names with their city
DROP VIEW vw_branch_addr;
```
(An UPDATE on `CITY` works because ADDR_DTLS is key-preserved; UPDATEs on `BRANCH_NO` fail with ORA-01779 — "maps to a non key-preserved table".)

#### Gotchas / Edge Cases
- Key preservation is about *keys*, not about where columns came from: one side of the join must carry its primary key into the view.
- ORA-01779 vs ORA-01752 vs ORA-01732 each mean a different view rule — the interview-safe statement: "not possible unless exactly one key-preserved table."
- Oracle errors aside, most production systems treat joined views as read-only. 

---

### CLUSTERS — Co-Located Tables
#### Explain It
A cluster physically stores rows of *different* tables that share a key in the **same data blocks**, so a join on the cluster key does one I/O instead of two. Clusters reduce disk I/O and speed up joins, but they complicate inserts and are a bad fit for columns that change often. In practice modern engines rarely need them; know the idea and its trade-off.

#### Prove It
```sql
CREATE CLUSTER cl_emp_branch (BRANCH_NO VARCHAR2(10));
DROP CLUSTER cl_emp_branch;   -- only possible while no tables live in it
```

#### Gotchas / Edge Cases
- A cluster key column must have matching types across the clustered tables.
- Clusters sacrifice DML simplicity for read speed — "columns updated often are not good candidates".

---

### SEQUENCES — Generated Keys
#### Explain It
A sequence generates unique numeric values for primary keys and audit numbers, with increment/min/max/cycle/cache controls. `NEXTVAL` yields the next value (advancing the sequence); `CURRVAL` re-reads the current value for the session. Sequences survive concurrent writers — two sessions never get the same value — and can be used in INSERT ... VALUES, and even in an UPDATE's SET list.

#### Prove It
```sql
CREATE SEQUENCE seq_custno START WITH 1 INCREMENT BY 1;
SELECT seq_custno.NEXTVAL FROM DUAL;     -- 1
SELECT seq_custno.CURRVAL FROM DUAL;     -- 1 (same session)
ALTER SEQUENCE seq_custno INCREMENT BY 2;
DROP SEQUENCE seq_custno;
```
```sql
-- populating a key column via a sequence
CREATE SEQUENCE SEQ_CUSTNO2 START WITH 1 INCREMENT BY 1;
CREATE TABLE CUSTOMERS_KEY (CUST_NO NUMBER, NAME VARCHAR2(20));
INSERT INTO CUSTOMERS_KEY VALUES (SEQ_CUSTNO2.NEXTVAL, 'Ivan');
INSERT INTO CUSTOMERS_KEY VALUES (SEQ_CUSTNO2.NEXTVAL, 'Nelson');
SELECT * FROM CUSTOMERS_KEY;             -- 1, 2
```

#### Gotchas / Edge Cases
- In Oracle, truncating/rolling back does NOT rewind a sequence: values are ever-increasing even if unused — saying "sequences are non-transactional" is a top interview one-liner.
- `CURRVAL` before any `NEXTVAL` in the session raises ORA-08002 — call NEXTVAL once first.
- Postgres uses `SERIAL`/`IDENTITY` (its own auto-increment), MySQL `AUTO_INCREMENT` — same job, different spellings.

---

### SNAPSHOTS / MATERIALIZED VIEWS — Pre-Computed Copies
#### Explain It
A snapshot (the classic Oracle name for what is now a materialized view) is a **stored copy** of a query result, refreshed at intervals — unlike a plain view, it has real data. Use it for fast reads on slow/distributed sources and for pre-aggregated summaries. Refresh modes: COMPLETE (rebuild), FAST (incremental), FORCE (fast if possible). The old `CREATE SNAPSHOT` syntax still works in Oracle 23c as a synonym for `CREATE MATERIALIZED VIEW`.

#### Prove It
```sql
CREATE SNAPSHOT snap_branch AS SELECT * FROM BRANCH_MSTR;   -- works in modern Oracle too
SELECT COUNT(*) FROM snap_branch;
DROP SNAPSHOT snap_branch;
```
```sql
CREATE MATERIALIZED VIEW mv_branch AS SELECT * FROM BRANCH_MSTR;
SELECT COUNT(*) FROM mv_branch;   -- reads the copy, not the base table
DROP MATERIALIZED VIEW mv_branch;
```

#### Gotchas / Edge Cases
- A snapshot can go STALE: it only refreshes at your schedule — the "why does my report show old data?" answer.
- Fast refresh needs a materialized-view log on the base table; without it Oracle falls back to COMPLETE rebuilds.
- Don't confuse snapshot (copies data) with view (copies nothing) — the pairing is a common question.

---

### Deleting Duplicate Rows via ROWID
#### Explain It
Because ROWID uniquely addresses every row, duplicates vanish by keeping the row with the *minimum* ROWID per logical group and deleting the rest. The idiom: `DELETE WHERE ROWID NOT IN (SELECT MIN(ROWID) ... GROUP BY <dup columns>)`.

#### Prove It
```sql
CREATE TABLE EMP_DUP AS SELECT * FROM EMP_MSTR;   -- start from a copy...
INSERT INTO EMP_DUP SELECT * FROM EMP_MSTR;       -- ...and double it
SELECT COUNT(*) FROM EMP_DUP;                     -- 14

DELETE FROM EMP_DUP WHERE ROWID NOT IN (
  SELECT MIN(ROWID) FROM EMP_DUP GROUP BY EMP_NO, FNAME, DEPT
);
SELECT COUNT(*) FROM EMP_DUP;                     -- 7: one row per group
```

#### Gotchas / Edge Cases
- The GROUP BY must list every column that defines "duplicate"; missing one keeps pseudo-duplicates.
- ROWIDs are Oracle-only — PostgreSQL uses `ctid`, and modern Oracle prefers a window-function `ROW_NUMBER()` approach; the ROWID idiom is a classic-but-Oracle answer.
- Never rely on "duplicate = same values" without also considering NULLs in the grouping columns.

---

### ROWNUM and the ORDER BY Trap
#### Explain It
`ROWNUM` numbers each row **as it is returned** — before `ORDER BY` runs. So `WHERE ROWNUM <= 3` gives three arbitrary rows (usually in fetch order), NOT "the top 3 by salary". To get top-N by a sort key you must sort first inside a subquery and number the sorted result outside. Also `WHERE ROWNUM > 5` matches zero rows because the first row gets ROWNUM 1, fails the test, and the numbering never advances. (The older claim that "ROWNUM is affected by ORDER BY only if an index exists" is disputed folklore — the reliable statement is the subquery-wrap above, or simply use `FETCH FIRST` from `sql-cmd.md`.)

#### Prove It
```sql
SELECT ROWNUM, BRANCH_NO FROM BRANCH_MSTR WHERE ROWNUM < 4;   -- first 3 fetched rows
SELECT ROWNUM FROM ORDERS WHERE ROWNUM > 5;                   -- 0 rows, always
-- top-2 oldest customers, correctly:
SELECT ROWNUM, FIRST_NAME FROM (
  SELECT * FROM CUSTOMERS ORDER BY AGE DESC
) WHERE ROWNUM < 3;
```

#### Gotchas / Edge Cases
- `ROWNUM = N` only ever matches N = 1; any other exact value returns nothing.
- Oracle 12c's `FETCH FIRST n ROWS ONLY` replaces this whole dance — prefer it; understand ROWNUM only to read legacy code.
- ROWNUM is assigned before window functions — `ROW_NUMBER()` is the modern, deterministic equivalent.