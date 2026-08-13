# Advanced Features in SQL\*Plus and Oracle

> **Dialect note:** Oracle-only (CONNECT BY, DECODE, ROWNUM, NVL, SOUNDEX, DUMP). Several snippets need the SQL\*Plus client, not just a SQL engine.

### Hierarchical (Tree-Structured) Queries
#### Explain It
`CONNECT BY` walks a tree relationship inside one table — the employee/manager example: each row links to a parent via a self-referencing key. `START WITH` names the root row(s), `PRIOR` tells Oracle which side is the parent, and `LEVEL` gives each row its depth in the tree. The classic use is organization charts; a plain self-join only goes one level deep, this goes all the way.

#### Prove It
```sql
SELECT LPAD(' ', LEVEL * 4) || FNAME || ' ' || LNAME "Employee Hierarchy"
FROM EMP_MSTR
CONNECT BY PRIOR EMP_NO = MNGR_NO
START WITH MNGR_NO IS NULL;
```

#### Gotchas / Edge Cases
- `PRIOR` placement is easy to invert: `CONNECT BY PRIOR EMP_NO = MNGR_NO` walks top-down from manager to reports; swap the sides to walk bottom-up.
- A cycle in the data (A reports to B who reports to A) makes the query loop — Oracle stops with `ORA-01436: CONNECT BY loop in user data`; the `NOCYCLE` keyword + `CONNECT_BY_ISCYCLE` handle it.
- CONNECT BY is Oracle-only; PostgreSQL's equivalent is a recursive CTE (`WITH RECURSIVE`), MySQL 8+ also recursive CTEs. Say "recursive CTE" if asked how to do it elsewhere.

---

### Trigger Pseudo-Records: :OLD and :NEW
#### Explain It
Inside a **row-level trigger**, Oracle exposes two implicit record variables: `:NEW` (the values the row will have *after* the DML) and `:OLD` (the values the row had *before* the DML). They are read/write for `:NEW` in `BEFORE` triggers, read-only in `AFTER` triggers; `:OLD` is always read-only. `INSERT` has `:NEW` only; `DELETE` has `:OLD` only; `UPDATE` has both.

SQL Server calls these `INSERTED` and `DELETED` pseudo-tables (relationally, not record-wise); the concept is identical.

#### Prove It
```sql
-- BEFORE INSERT trigger: enforce a business rule on :NEW
CREATE OR REPLACE TRIGGER trg_emp_sal_chk
  BEFORE INSERT OR UPDATE OF sal ON emp_mstr
  FOR EACH ROW
BEGIN
  IF :NEW.sal < 20000 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Salary below minimum threshold');
  END IF;
END;
/

-- AFTER UPDATE trigger: audit changes using :OLD and :NEW
CREATE OR REPLACE TRIGGER trg_emp_audit
  AFTER UPDATE ON emp_mstr
  FOR EACH ROW
BEGIN
  INSERT INTO emp_audit_log
    (emp_no, old_sal, new_sal, changed_at)
  VALUES
    (:OLD.emp_no, :OLD.sal, :NEW.sal, SYSDATE);
END;
/

-- Demonstrate the trigger
UPDATE emp_mstr SET sal = 25000 WHERE emp_no = 'E1';
-- emp_audit_log now contains (E1, old_sal, 25000, SYSDATE)
```

#### Gotchas / Edge Cases
- `:NEW.column_name` in a `BEFORE` trigger is **writable** — changing `:NEW.sal := 30000` silently overrides the incoming value before the row is stored.
- `:OLD` is **always read-only** — trying `:OLD.sal := 30000` raises `ORA-04084: cannot change NEW values for this trigger type`.
- Statement-level triggers (no `FOR EACH ROW`) have no `:OLD`/`:NEW` — they fire once per statement, not once per row.
- `INSERTED`/`DELETED` in SQL Server are *tables* (potentially multi-row); `:OLD`/`:NEW` in Oracle are *records* (one row per trigger execution) — this structural difference matters when porting trigger logic.

---

### Matrix Reports with DECODE
#### Explain It
`DECODE(expr, val1, out1, val2, out2, ..., default)` returns the output matching the first equal value — a CASE-expression in function form. Pairing `SUM(DECODE(...,1,0))` with GROUP BY turns rows into a cross-tab: each DECODE contributes 1 when a row belongs to that column's bucket, 0 otherwise, so the SUM is the column's count.

#### Prove It
```sql
-- employees per branch, one output column per branch
SELECT B.NAME "BRANCH",
       SUM(DECODE(E.BRANCH_NO,'B1',1,0)) "B1",
       SUM(DECODE(E.BRANCH_NO,'B2',1,0)) "B2",
       SUM(DECODE(E.BRANCH_NO,'B3',1,0)) "B3",
       COUNT(E.EMP_NO) "TOTAL"
FROM EMP_MSTR E JOIN BRANCH_MSTR B ON B.BRANCH_NO = E.BRANCH_NO
GROUP BY B.NAME;
```
```sql
-- per-customer account-product matrix
SELECT CUST_NO,
       SUM(DECODE(SUBSTR(ACCT_FD_NO,1,2),'CA',1,0)) "CURRENT ACCOUNTS",
       SUM(DECODE(SUBSTR(ACCT_FD_NO,1,2),'SB',1,0)) "SAVINGS ACCOUNTS",
       SUM(DECODE(SUBSTR(ACCT_FD_NO,1,2),'FS',1,0)) "FIXED DEPOSITS",
       COUNT(ACCT_FD_NO) "TOTAL"
FROM ACCT_FD_CUST_DTLS
GROUP BY CUST_NO;
```

#### Gotchas / Edge Cases
- DECODE works on **equality only** — for ranges/`>` use `CASE WHEN`, the ANSI portable form (usually the better interview answer).
- The trailing default matters: without it, non-matching values yield NULL, and `SUM(NULL + 1)` arithmetic breaks.
- DECODE's matching treats NULLs as equal to NULLs (DECODE(NULL, NULL, 'x') → 'x'), unlike plain `=` comparison — a subtle trap.

---

### Peeking at Stored Values: DUMP
#### Explain It
`DUMP` shows how Oracle really stores a value: the internal type, byte length, and the byte values. It is a debugging/teaching tool — useful to prove why `'1'` differs from `1`, or why CHAR pads and VARCHAR2 doesn't.

#### Prove It
```sql
SELECT DUMP(ACCT_NO) FROM ACCT_MSTR WHERE ROWNUM < 2;
-- e.g. Typ=1 Len=3: 83,66,49      (the bytes of 'SB1')
```

#### Gotchas / Edge Cases
- DUMP output format varies by character set (AL32UTF8 vs WE8MSWIN1252) — byte numbers differ across databases even for the same text.
- `DUMP` is diagnostic, not part of normal queries; don't reach for it in interviews beyond "it shows internal representation."

---

### Every Nth Row / Rows X to Y (ROWNUM)
#### Explain It
Because ROWNUM numbers rows as they are fetched, math on it can pick a pattern: `MOD(ROWNUM, 2) = 0` selects the even-numbered rows. To grab a *window* of rows (e.g. the 4th–7th), number the rows in a subquery first and filter on that inner number — the pattern works because the subquery already assigned RN before the outer WHERE runs.

#### Prove It
```sql
SELECT ROWNUM, EMP_NO, FNAME FROM EMP_MSTR WHERE MOD(ROWNUM, 2) = 0;   -- even rows

SELECT * FROM (SELECT ROWNUM RN, FNAME FROM EMP_MSTR WHERE ROWNUM < 8)
WHERE RN BETWEEN 4 AND 7;                                              -- rows 4..7
```

#### Gotchas / Edge Cases
- ROWNUM is assigned before ORDER BY — "even rows" without ordering is a fetch-order pattern, not a logical one (see `advanced-sql.md` for the full trap).
- The `MOD(ROWNUM, 2) = 0` trick stops working after the first non-matching row in some engines — Oracle keeps assigning until done, but don't rely on the pattern for correctness.

---

### Generate Primary Keys with Sequences
#### Explain It
Oracle sequences are the standard key generator: `NEXTVAL` produces a new unique number per call, safe under concurrency. Sequence values can be concatenated with text (e.g. `'C' || seq.NEXTVAL`) to make human-readable keys, and they are non-transactional — rolled-back statements burn numbers.

#### Prove It
```sql
CREATE SEQUENCE SEQ_CUSTNO START WITH 1 INCREMENT BY 1;
CREATE TABLE CUSTOMERS_KEY (CUST_NO VARCHAR2(10), NAME VARCHAR2(20));
INSERT INTO CUSTOMERS_KEY VALUES ('C' || SEQ_CUSTNO.NEXTVAL, 'Ivan');     -- C1
INSERT INTO CUSTOMERS_KEY VALUES ('C' || SEQ_CUSTNO.NEXTVAL, 'Nelson');   -- C2
SELECT * FROM CUSTOMERS_KEY;
```
(Plain numeric keys work the same: `INSERT ... VALUES (SEQ_CUSTNO.NEXTVAL, ...)`.)

#### Gotchas / Edge Cases
- Gaps are normal: ROLLBACK or DELETE consumed numbers are never reused — "sequences are not gap-free" is the interview one-liner.
- `CURRVAL` without a session's prior `NEXTVAL` raises ORA-08002.
- Oracle 12c+ has `IDENTITY` columns as the declarative alternative; sequences remain the portable explanation.

---

### Date Arithmetic and Formatting
#### Explain It
Oracle dates are numeric: adding `1` adds one day, `1/24` one hour, `1/(24*60)` one minute. `TO_CHAR(..., 'fmt')` renders any date part. This is the fastest way to show "I know Oracle dates" in a coding question.

#### Prove It
```sql
SELECT SYSDATE + 1    FROM DUAL;    -- tomorrow
SELECT SYSDATE + 1/24 FROM DUAL;    -- one hour from now
SELECT TO_CHAR(SYSDATE, 'Month DD, YYYY') FROM DUAL;  -- e.g. August 13, 2026
```

#### Gotchas / Edge Cases
- `SYSDATE + 0.5` is noon tomorrow — fractional days are hours; forgetting `1/24` is the classic off-by-24 bug (see `computations.md`).
- Format masks are case-sensitive in meaning ('Month' vs 'MONTH' vs 'MON') — each yields a different shape of text.

---

### NULL Handling: NVL and Line Feeds
#### Explain It
`NVL(value, replacement)` returns the replacement when the value is NULL — the two-argument special case of COALESCE. `CHR(10)` is the newline character, useful for composing multi-line messages in one SELECT.

#### Prove It
```sql
SELECT NVL(FNAME, 'A'), NVL(MNAME, 'Corporate'), NVL(LNAME, 'Customer')
FROM CUST_MSTR WHERE ROWNUM < 2;

SELECT 'CUSTOMER NAME: ' || FNAME || CHR(10) ||
       'BIRTHDATE: ' || TO_CHAR(DOB_INC, 'DD-MON-YYYY')
FROM CUST_MSTR WHERE ROWNUM < 2;
```

#### Gotchas / Edge Cases
- Oracle's `''` IS NULL, so NVL('') hits the replacement; in PostgreSQL `''` is a real string and passes through — the perennial dialect difference.
- `NVL` evaluates its second argument even when unnecessary; `COALESCE` short-circuits (see `computations.md`).

---

### Soundex — Phonetic Matching
#### Explain It
`SOUNDEX` converts a word into a phonetic code so that names that *sound* alike but are spelled differently (Ivan / Ivon) compare equal. It is a rough fuzzy-search tool, useful in customer-service lookups and dedup, and obviously not usable for semantics.

#### Prove It
```sql
SELECT * FROM CUST_MSTR WHERE SOUNDEX(FNAME) = SOUNDEX('Ivan');
```

#### Gotchas / Edge Cases
- Soundex works on English pronunciation only — it degrades for non-English names and is useless for numbers.
- The code is a letter + three digits; common sound-alikes can still collide (false positives are expected).

---

### Numeric-to-Words (Julian 'JSP')
#### Explain It
`TO_CHAR(n, 'JSP')` spells a number in English words by first converting the number to a Julian date format ('J') and then spelling it ('SP' = spell). It is a fun one-liner for cheque-printing examples.

#### Prove It
```sql
SELECT TO_CHAR(TO_DATE(34654,'J'),'JSP') FROM DUAL;
-- THIRTY-FOUR THOUSAND SIX HUNDRED FIFTY-FOUR
```

#### Gotchas / Edge Cases
- The 'J' trick is limited by Oracle's Julian-date range — huge numbers overflow; don't use it in production.
- 'SP' spelling inherits NLS language settings ('SP' vs 'SPTH' ordinal) — a format-model curiosity more than a daily tool.

---

### Changing an Oracle Password (ALTER USER)
#### Explain It
Passwords are changed with `ALTER USER ... IDENTIFIED BY ...`, run by the user or the DBA for others. Scripts should never hardcode it; it exists in this notes file as a "how do I reset a password" SRE-ask.

#### Prove It
```sql
-- as DBA, or as the user themselves:
ALTER USER hansel IDENTIFIED BY hansel123;
```

#### Gotchas / Edge Cases
- A user can alter their own password; a DBA can alter anyone's (and even expire it: `ALTER USER x PASSWORD EXPIRE;`).
- In newer Oracle versions, `IDENTIFIED BY` triggers password-verification-function rules — a too-simple password can be rejected.

---

### Drop/Rename Workarounds (Historical)
#### Explain It
Before Oracle 8i you could not `DROP COLUMN`, and renaming a column had no direct command. The old workarounds — copy the table without the column, drop the old one, rename the new — are still useful to *know* because they explain why "drop column" is a table rebuild under the hood, and they resurrect when a production table won't let you drop (referential constraints).

#### Prove It
```sql
-- the 8i-era recipe for "drop column":
CREATE TABLE B_NEW AS SELECT BRANCH_NO, NAME FROM BRANCH_MSTR;  -- omit the column
DROP TABLE BRANCH_MSTR;
RENAME B_NEW TO BRANCH_MSTR;
```
(Modern Oracle: `ALTER TABLE t DROP COLUMN c;` — and `ALTER TABLE t RENAME COLUMN a TO b;` — from `sql-cmd.md`.)

#### Gotchas / Edge Cases
- The copy-and-rename recipe silently loses constraints, indexes, and grants — always re-add them.
- Renaming invalidates dependent views/procedures until recompiled (see `db-obj.md`).

---

### CSV Output in SQL\*Plus
#### Explain It
These are SQL\*Plus *client* settings that shape what gets printed: `SET COLSEP ','` puts commas between columns, `SPOOL file` writes output to a file, `SPOOL OFF` closes it. The same trio (headers suppressing, trimspool, colsep) is the classic "export to CSV" recipe.

#### Prove It
```sql
SET COLSEP ','
SPOOL MY_EMP_REPORT.TXT
SELECT BRANCH_NO, NAME FROM BRANCH_MSTR;
SPOOL OFF
```

#### Gotchas / Edge Cases
- These are client commands, NOT SQL — they fail in JDBC/other tools; put them in a `.sql` script run from SQL\*Plus.
- For real CSV use `SET HEADING OFF` + `SET TRIMSPOOL ON`; without them the file gets column headers and trailing blanks.
- SQLcl/`SELECT ... FOR CSV` (or `\spool`-style tools) are modern alternatives — mention SQLcl if asked.