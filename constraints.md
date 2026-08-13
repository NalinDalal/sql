# Data Constraints

> **Dialect note:** Oracle (VARCHAR2, `ON DELETE` rules). Chapter 8 of the course PDF (pp. 137–155).

### What Constraints Are
#### Explain It
Constraints are business rules enforced by the database engine *before* data is stored. When an INSERT or UPDATE arrives, the engine checks the row against every active constraint; if it passes, the row is stored, otherwise the statement is rejected with an error. They are declared with `CREATE TABLE` or added later with `ALTER TABLE`. Constraints split into I/O-style referential ones (PRIMARY KEY, FOREIGN KEY) and business-rule ones (NOT NULL, UNIQUE, CHECK).

#### Prove It
```sql
CREATE TABLE CUST_MSTR2 (
  CUST_NO VARCHAR2(10) PRIMARY KEY,
  FNAME   VARCHAR2(25) NOT NULL
);
```

#### Gotchas / Edge Cases
- Constraints exist for data *integrity*, not performance — but the engine usually creates an index behind PK/UNIQUE (see `advanced-sql.md`).
- A row can be rejected by exactly one constraint at a time — Oracle reports the first failure (e.g. `ORA-00001`, `ORA-01400`, `ORA-02290`); debugging means fixing them one by one.

---

### PRIMARY KEY
#### Explain It
A primary key is a column (or set of columns — a *composite* key) that uniquely identifies every row. It combines **NOT NULL + UNIQUE**: no duplicate values, no NULLs. One table can have only one primary key, and Oracle builds a unique index behind it automatically. It is not compulsory, but it is the backbone of relations between tables.

#### Prove It
```sql
-- single-column PK, column level
CREATE TABLE CUST_MSTR2 (
  CUST_NO VARCHAR2(10) PRIMARY KEY,
  FNAME   VARCHAR2(25),
  MNAME   VARCHAR2(25),
  LNAME   VARCHAR2(25)
);

-- composite PK, table level (up to 16 columns in Oracle)
CREATE TABLE FD_MSTR3 (
  FD_SER_NO    VARCHAR2(10),
  CORP_CUST_NO VARCHAR2(10),
  AMT          NUMBER(8,2),
  CONSTRAINT pk_fd3 PRIMARY KEY (FD_SER_NO, CORP_CUST_NO)
);

INSERT INTO FD_MSTR3 VALUES ('FS1','C1',100);
INSERT INTO FD_MSTR3 VALUES ('FS1','C1',200);   -- ORA-00001: unique constraint violated
```

#### Gotchas / Edge Cases
- No part of a composite PK may be NULL — a partial-NULL insert is rejected (ORA-01400).
- A primary key column cannot be LONG/LONG RAW, and you get exactly one PK per table.
- "PK = UNIQUE + NOT NULL" is the standard interview shorthand; UNIQUE alone allows NULLs (next concept).
- Order of columns in a composite key matters for index-usage: a query filtering only the *second* column can't use the composite index efficiently (see indexes in `advanced-sql.md`).

---

### UNIQUE
#### Explain It
A UNIQUE constraint forbids duplicate values in a column or column set, but — unlike the primary key — it **permits multiple NULLs**. Oracle treats each NULL as "no value", so NULLs never conflict with each other. A table may have several UNIQUE columns, and each gets a unique index automatically.

#### Prove It
```sql
CREATE TABLE CUST_MSTR2 (
  CUST_NO VARCHAR2(10) PRIMARY KEY,
  FNAME   VARCHAR2(25) NOT NULL,
  LNAME   VARCHAR2(25) UNIQUE          -- table level: UNIQUE (FNAME, LNAME)
);

INSERT INTO CUST_MSTR2 VALUES ('C2', 'Nelson', NULL);
INSERT INTO CUST_MSTR2 VALUES ('C3', 'Maya',   NULL);  -- multiple NULLs: allowed
INSERT INTO CUST_MSTR2 VALUES ('C4', 'Dup',    'Bayross'); -- ORA-00001: duplicate rejected
```

#### Gotchas / Edge Cases
- "How many NULLs does UNIQUE allow?" — multiple, in Oracle/Postgres/MySQL; the SQL standard leaves it vendor-defined, so answer "Oracle: many, because NULL != NULL" and note the dialect dependence.
- This is THE difference between PRIMARY KEY and UNIQUE, and it is the most-asked constraints interview question.
- UNIQUE also works as a composite (up to 16 columns in Oracle) and cannot use LONG/LONG RAW.

---

### FOREIGN KEY
#### Explain It
A foreign key is a column (or group) whose values must exist as a PRIMARY KEY or UNIQUE value in another (or the *same*) table. The table holding the FK is the child/detail table; the referenced one is the parent/master table. The parent record cannot be deleted or its key changed while child rows reference it — unless you declare `ON DELETE CASCADE` (delete the children too) or `ON DELETE SET NULL` (blank the child's FK).

#### Prove It
```sql
-- column level: EMP_MSTR.BRANCH_NO must exist in BRANCH_MSTR
CREATE TABLE EMP_MSTR2 (
  EMP_NO    VARCHAR2(10) PRIMARY KEY,
  BRANCH_NO VARCHAR2(10) REFERENCES BRANCH_MSTR(BRANCH_NO)
);
INSERT INTO EMP_MSTR2 VALUES ('E1', 'B1');   -- ok
INSERT INTO EMP_MSTR2 VALUES ('E9', 'B99');  -- ORA-02291: parent key not found

-- table level + cascade
CREATE TABLE FD_DTLS2 (
  FD_SER_NO VARCHAR2(10),
  FD_NO     VARCHAR2(10),
  AMT       NUMBER(8,2),
  CONSTRAINT fk_fd2 FOREIGN KEY (FD_SER_NO) REFERENCES FD_MSTR2(FD_SER_NO)
    ON DELETE CASCADE
);
DELETE FROM FD_MSTR2 WHERE FD_SER_NO = 'FS1';  -- children deleted automatically

-- SET NULL variant
CREATE TABLE FD_DTLS3 (
  FD_SER_NO VARCHAR2(10),
  AMT       NUMBER(8,2),
  CONSTRAINT fk_fd3 FOREIGN KEY (FD_SER_NO) REFERENCES FD_MSTR2(FD_SER_NO)
    ON DELETE SET NULL
);
DELETE FROM FD_MSTR2 WHERE FD_SER_NO = 'FS2';  -- child FD_SER_NO becomes NULL
```

#### Gotchas / Edge Cases
- The referenced column must be PRIMARY KEY or UNIQUE; data types must match.
- The child side may contain NULLs and duplicates — only *present* values must exist in the parent.
- **Oracle has no `ON UPDATE CASCADE`** (unlike MySQL/Postgres): changing a parent key requires manual updates or delete+reinsert. A favorite dialect trap.
- Without any `ON DELETE` clause, Oracle's default is to block the parent delete when children exist (ORA-02292).
- A self-referencing FK is how employee→manager hierarchies are modeled (see SELF JOIN in `joins.md`).

---

### NOT NULL and the NULL Concept
#### Explain It
`NULL` means "unknown / not applicable" — it is *not* the empty string and *not* zero. A NOT NULL constraint makes a column mandatory: the row is rejected if no value is supplied. Notice that in Oracle, an empty string `''` is treated exactly like NULL, so `''` also fails a NOT NULL column.

#### Prove It
```sql
CREATE TABLE NULL_PROBE (ID NUMBER, NOTE VARCHAR2(10));
INSERT INTO NULL_PROBE VALUES (1, NULL);   -- allowed: NOTE is nullable
SELECT * FROM NULL_PROBE;                  -- NOTE displays blank (NULL)

CREATE TABLE CUST_MSTR4 (
  CUST_NO VARCHAR2(10) PRIMARY KEY,
  FNAME   VARCHAR2(25) NOT NULL
);
INSERT INTO CUST_MSTR4 (CUST_NO, FNAME) VALUES ('C5', NULL);  -- ORA-01400: cannot insert NULL
```

#### Gotchas / Edge Cases
- `WHERE col = NULL` never matches; use `IS NULL` (full treatment in `10-interview-questions.md`).
- Oracle collapses `''` to NULL — in PostgreSQL `''` is a real empty string. A classic cross-dialect interview surprise.
- PRIMARY KEY columns are implicitly NOT NULL; UNIQUE columns are not.
- NULL propagates through arithmetic: `NULL + 1000` is `NULL` (see `computations.md`).

---

### CHECK
#### Explain It
A CHECK constraint validates a row with a Boolean expression that must evaluate to TRUE (NULL results are accepted — "unknown" doesn't fail). It is the engine-level version of application validation: format rules (`CUST_NO LIKE 'C%'`), domain rules (`AMT > 0`), cross-column rules (`start < end`). It can be declared at column or table level, and named for readable errors.

#### Prove It
```sql
CREATE TABLE CUST_MSTR3 (
  CUST_NO VARCHAR2(10),
  FNAME   VARCHAR2(25),
  MNAME   VARCHAR2(25),
  LNAME   VARCHAR2(25),
  CONSTRAINT chk_cust CHECK (CUST_NO LIKE 'C%'),
  CONSTRAINT chk_fn   CHECK (FNAME = UPPER(FNAME))
);
INSERT INTO CUST_MSTR3 VALUES ('C1', 'IVAN', 'N', 'BAYROSS');  -- ok
INSERT INTO CUST_MSTR3 VALUES ('X1', 'IVAN', 'N', 'BAYROSS');  -- ORA-02290: CHECK violated
```

#### Gotchas / Edge Cases
- Oracle CHECK conditions **cannot** contain subqueries, sequences, or the pseudocolumns `SYSDATE`, `UID`, `USER`, `USERENV` — it must be evaluable from the row's own values alone.
- A CHECK that evaluates to NULL (e.g. any row where the checked column is NULL) **passes** — "fail" only happens on FALSE. This trips people up in interviews.
- CHECKs are per-row: they never compare against other rows or tables — that job belongs to FK/triggers.

---

### Column-Level vs Table-Level Constraints
#### Explain It
A constraint is *column-level* when written inline with its column's definition, and *table-level* when written after all columns. Column-level is fine for single-column rules (NOT NULL, a UNIQUE column); table-level is required for composite keys (multi-column PK/FK/UNIQUE/CHECK) and for naming your constraints via `CONSTRAINT name ...`.

#### Prove It
```sql
-- column level
CREATE TABLE T1 (ID NUMBER PRIMARY KEY, EMAIL VARCHAR2(30) UNIQUE);

-- table level (needed for the composite PK; also lets you name it)
CREATE TABLE T2 (
  A VARCHAR2(10),
  B VARCHAR2(10),
  CONSTRAINT pk_t2 PRIMARY KEY (A, B)
);
```

#### Gotchas / Edge Cases
- NOT NULL can only be column-level in classic Oracle (12c+ allows table-level `NOT NULL` with new syntax — don't rely on it).
- Named constraints produce readable error/validation messages and can be dropped by name (`ALTER TABLE t DROP CONSTRAINT pk_t2;`); unnamed ones get auto-generated names like `SYS_C008877`.
- "Column vs table level" is a stock interview question — the short answer is composite keys force table-level.