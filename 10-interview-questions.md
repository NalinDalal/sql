# 10 SQL Interview Questions

> **Dialect note:** All code is Oracle/PL-SQL unless stated. Where vendor differences matter, they are noted inline. For deeper coverage of each topic, follow the links to the relevant module file.

---

### Question 1 — Normalisation Forms: 1NF → BCNF

**"Walk me through the normalisation forms. What is BCNF and why does it matter?"**

Normalisation removes redundancy and prevents anomalies. The key forms every candidate should name in order:

| Form    | Rule                                                       |
| ------- | ---------------------------------------------------------- |
| 1NF     | Atomic values only; no repeating groups                     |
| 2NF     | 1NF + no partial dependency on a composite PK               |
| 3NF     | 2NF + no transitive dependency (non-key → non-key)          |
| BCNF    | Every determinant must be a candidate key (stricter than 3NF) |

**BCNF example** — a `course_teachers` table where both `course_id` and `teacher_id` determine each other. Neither is a sole PK, so a transitive loop exists. BCNF requires splitting into two tables: `courses` and `teachers`.

```sql
-- Before BCNF (violates BCNF: neither col is a candidate key on its own)
-- course_teachers(course_id, teacher_id, ...)

-- After BCNF: split into two tables
CREATE TABLE courses (
  course_id   VARCHAR2(10) PRIMARY KEY,
  course_name VARCHAR2(100)
);

CREATE TABLE teachers (
  teacher_id  VARCHAR2(10) PRIMARY KEY,
  teacher_name VARCHAR2(100)
);

-- Link table (now both FKs reference PKs)
CREATE TABLE course_teachers (
  course_id  VARCHAR2(10) REFERENCES courses(course_id),
  teacher_id VARCHAR2(10) REFERENCES teachers(teacher_id),
  PRIMARY KEY (course_id, teacher_id)
);
```

**Why this is asked:** Normalisation is a database design fundamentals question. BCNF is the upper bound most interviewers expect a senior candidate to know. Stopping at 3NF without mentioning BCNF signals a gap.

> Full normalisation forms with runnable examples: `1.intro.md`

---

### Question 2 — FULL OUTER JOIN

**"What does FULL OUTER JOIN return? How is it different from UNION of LEFT and RIGHT joins?"**

`FULL OUTER JOIN` returns all rows from both tables — matched rows combined, plus unmatched rows from each side padded with `NULL`. It is equivalent to `LEFT JOIN UNION RIGHT JOIN` in result set (both produce the same rows), but `FULL OUTER JOIN` is a **single pass** and often more readable.

```sql
-- All customers, all accounts — including customers with no accounts
-- and accounts with no matching customer (shouldn't happen in a proper FK design,
-- but FULL JOIN surfaces orphaned data)
SELECT c.cust_id, c.cust_name, a.acct_no, a.acct_type
  FROM cust_mstr c
  FULL OUTER JOIN acct_mstr a ON c.cust_id = a.cust_id
 ORDER BY c.cust_id;
```

**Why this is asked:** Many candidates know `INNER` and `LEFT` but skip `FULL`. Interviewers use it to test completeness of join knowledge. It is also the right tool for "find orphans in either table" questions.

> Full join types with summary table and Mermaid diagrams: `joins.md`

---

### Question 3 — B-tree Index: What It Is and When NOT to Use One

**"What is a B-tree index? When would you deliberately not create one?"**

A B-tree (Balanced Tree) is the default index structure in Oracle (and most databases). It stores sorted key values with leaf pages containing rowids, enabling `O(log n)` point lookups and range scans. Oracle uses a **B*-tree variant** — leaf pages are linked in a doubly-linked list for efficient range traversal.

**When NOT to index:**
- **Low-cardinality columns** — a column with only two distinct values (`Y/N`, `M/F`) gives the optimizer little selectivity benefit; the index is likely ignored and adds write overhead
- **High-write tables** — every `INSERT`/`UPDATE`/`DELETE` must maintain the index; on a table with 10M+ writes/day, the index overhead can exceed its read benefit
- **Small tables** — a full table scan is cheaper than an index traversal for a few hundred rows

```sql
-- Good: high-cardinality, frequently queried column
CREATE INDEX idx_acct_no ON acct_mstr(acct_no);

-- Bad: low-cardinality, adds write overhead with little read benefit
CREATE INDEX idx_active ON acct_mstr(is_active);  -- only 'Y'/'N'

-- Verify whether Oracle uses an index
EXPLAIN PLAN FOR
  SELECT * FROM acct_mstr WHERE acct_no = 'SB1';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
-- Look for INDEX RANGE SCAN vs FULL TABLE SCAN
```

**Why this is asked:** This separates junior from mid-level candidates. Anyone can say "indexes speed up queries" — the interview answer is knowing **when they slow you down**.

> Full index types with the worked index-damage example: `advanced-sql.md`

---

### Question 4 — Isolation Levels Across Vendors

**"What isolation levels does Oracle support? How do they compare to Postgres and MySQL?"**

Isolation levels control how much one transaction sees of another's uncommitted changes. Oracle supports only two:

| Level              | Oracle | Postgres | MySQL (InnoDB) |
| ------------------ | :----: | :------: | :------------: |
| READ UNCOMMITTED   |   —    |    ✓     |      ✓         |
| READ COMMITTED     |   ✓    |    ✓     |      ✓         |
| REPEATABLE READ    |   —    |    ✓     |   ✓ (default)  |
| SERIALIZABLE       |   ✓    |    ✓     |      ✓         |

Oracle defaults to **READ COMMITTED** — every query sees only committed data as of the query start time, but re-reading the same table within a transaction can return different rows if another session commits in between.

Oracle achieves multi-version concurrency via **undo segments** (read consistency), not via locks. A `SELECT` never blocks a writer, and a writer never blocks a reader.

```sql
-- Set isolation level at session level (Oracle)
ALTER SESSION SET ISOLATION_LEVEL = SERIALIZABLE;
-- All subsequent queries in this session see the database as of the
-- first SELECT time — phantom rows are prevented

-- Verify current isolation level
SELECT SYS_CONTEXT('USERENV', 'ISOLATION_LEVEL') FROM dual;
-- Returns 'READ COMMITTED' or 'SERIALIZABLE'

-- Postgres equivalent
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- MySQL equivalent
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

**Why this is asked:** Isolation levels are a core ACID concept. Knowing that Oracle **omits** `READ UNCOMMITTED` and `REPEATABLE READ` shows real multi-vendor experience — a common gap in candidates who only know one database.

> Full isolation-level matrix and anomalies: `transactions.md`

---

### Question 5 — PRIMARY KEY vs UNIQUE Constraint and NULL

**"What is the difference between PRIMARY KEY and UNIQUE? Can a UNIQUE column contain NULLs?"**

| Property            | PRIMARY KEY                    | UNIQUE                              |
| ------------------- | ------------------------------ | ----------------------------------- |
| NULL allowed        | No (implicitly `NOT NULL`)     | Yes — but only **one** NULL per column in Oracle |
| Count per table     | Exactly one                    | Zero or more                        |
| Implicit index      | Yes (unique index)             | Yes (unique index)                  |
| Purpose             | Identify rows                  | Enforce business uniqueness         |

Oracle enforces `UNIQUE` with a **unique B-tree index**. Multiple `NULL`s in a non-PK column: Oracle allows **only one** `NULL` in a `UNIQUE` column (treating `NULL != NULL`). Postgres allows **multiple** `NULL`s in a `UNIQUE` column by default.

```sql
-- PRIMARY KEY (NOT NULL + UNIQUE)
ALTER TABLE emp_mstr ADD CONSTRAINT emp_pk PRIMARY KEY (emp_id);

-- UNIQUE (allows one NULL in Oracle)
ALTER TABLE emp_mstr ADD CONSTRAINT emp_email_uk UNIQUE (email);

-- Demonstrate: inserting a second NULL into a UNIQUE column
-- First NULL — succeeds
INSERT INTO emp_mstr(emp_id, email) VALUES (999, NULL);
-- Second NULL — ORA-00001: unique constraint (EMP_EMAIL_UK) violated

-- Check constraints
SELECT constraint_name, constraint_type, search_condition
  FROM user_constraints
 WHERE table_name = 'EMP_MSTR';
```

**Why this is asked:** The `NULL` behaviour under `UNIQUE` is a classic trap. Many candidates assume `NULL` is unrestricted in a unique column. Knowing the Oracle/Postgres difference is interview gold.

> Full constraint types with runnable examples: `constraints.md`

---

### Question 6 — Stored Procedure vs Function

**"When would you use a stored procedure versus a function? Can a function be called from a SELECT statement?"**

| Aspect           | Procedure                    | Function                          |
| ---------------- | ---------------------------- | --------------------------------- |
| Return value     | Optional (`OUT` params)      | **Mandatory** — exactly one value |
| Callable from SQL| No (only `EXEC`/`CALL`)     | Yes (`SELECT`, `WHERE`, etc.)     |
| DML in body      | Allowed                      | Disallowed unless `AUTONOMOUS_TRANSACTION` |
| Use in           | Side effects, multi-row DML  | Computed values used in queries   |

```sql
-- Function: callable from SELECT, returns one value
CREATE OR REPLACE FUNCTION annual_salary(p_emp_id IN NUMBER) RETURN NUMBER AS
  v_sal NUMBER;
BEGIN
  SELECT sal INTO v_sal FROM emp_mstr WHERE emp_id = p_emp_id;
  RETURN v_sal * 12;
END;
/

SELECT emp_id, annual_salary(emp_id) AS annual_sal FROM emp_mstr;

-- Procedure: not callable from SQL; performs an action
CREATE OR REPLACE PROCEDURE apply_hike(p_emp_id IN NUMBER, p_pct IN NUMBER) AS
BEGIN
  UPDATE emp_mstr SET sal = sal * (1 + p_pct / 100)
   WHERE emp_id = p_emp_id;
  COMMIT;
END apply_hike;
/

-- Invoke procedure
EXEC apply_hike(101, 10);
-- Cannot do: SELECT apply_hike(101, 10) FROM dual;  ← raises error
```

**Why this is asked:** This tests whether the candidate understands the call-context boundary. Mixing them up in a design discussion signals a lack of production PL/SQL experience.

> Full procedure/function/package patterns with overloading: `db-obj.md`

---

### Question 7 — TRUNCATE vs DELETE

**"What is the difference between TRUNCATE and DELETE? Is TRUNCATE transaction-safe?"**

| Property          | TRUNCATE                         | DELETE                              |
| ----------------- | -------------------------------- | ------------------------------------ |
| Transaction-safe  | **Oracle: no** (implicit commit) | **Yes** in Oracle, Postgres, MySQL   |
| WHERE clause      | No                               | Yes                                  |
| Logging           | Minimal (deallocates extents)    | Row-by-row undo/redo                 |
| Triggers fired    | No                               | Yes (per row)                        |
| Identity reset    | Yes                              | No                                   |
| Speed             | Much faster                      | Slower (especially with many rows)   |

```sql
-- DELETE: transaction-safe, fires triggers, respects WHERE
DELETE FROM emp_mstr WHERE dept_id = 99;
ROLLBACK; -- rows are restored

-- TRUNCATE: implicit COMMIT in Oracle — cannot be rolled back
TRUNCATE TABLE emp_mstr;
-- The table is empty immediately; ROLLBACK does NOT restore rows

-- Postgres: TRUNCATE is transactional
BEGIN;
  TRUNCATE TABLE emp_mstr;
ROLLBACK; -- rows are restored (Postgres only, not Oracle)
```

**Why this is asked:** The `TRUNCATE` transaction behaviour is a **vendor difference** that surfaces in migrations and incident post-mortems. A candidate who knows the Oracle-specific implicit commit and contrasts it with Postgres shows real-world awareness.

> Full ACID and transaction control: `transactions.md`

---

### Question 8 — ROWNUM vs ROW_NUMBER()

**"What is ROWNUM? How does ORDER BY interact with it? What is the correct way to get the top-N rows?"**

`ROWNUM` is a **pseudo-column** assigned to rows **as they are retrieved** — before `ORDER BY` is applied. If you write `WHERE ROWNUM <= 5 ORDER BY sal DESC`, you get the first 5 rows in whatever order Oracle happens to read them, then sorts those 5 — not the top 5 salaries.

The correct pattern is a **subquery (inline view)**:

```sql
-- WRONG: ROWNUM assigned before ORDER BY — arbitrary 5 rows returned
SELECT emp_id, emp_name, sal FROM emp_mstr
 WHERE ROWNUM <= 5
 ORDER BY sal DESC;

-- CORRECT: sort first, then assign row numbers
SELECT * FROM (
  SELECT emp_id, emp_name, sal
    FROM emp_mstr
   ORDER BY sal DESC
) WHERE ROWNUM <= 5;

-- Best practice: ANSI ROW_NUMBER() (Oracle 12c+)
SELECT emp_id, emp_name, sal
  FROM (
    SELECT emp_id, emp_name, sal,
           ROW_NUMBER() OVER (ORDER BY sal DESC) AS rn
      FROM emp_mstr
  )
 WHERE rn <= 5;

-- ANSI FETCH FIRST (Oracle 12c+ — cleanest)
SELECT emp_id, emp_name, sal
  FROM emp_mstr
 ORDER BY sal DESC
 FETCH FIRST 5 ROWS ONLY;
```

**Why this is asked:** `ROWNUM` ordering is one of the most common SQL mistakes in production code. Interviewers test it because fixing it requires understanding the logical order of execution in a `SELECT` statement.

> ROWNUM vs ROWID, FETCH FIRST, and common traps: `advanced-sql.md`

---

### Question 9 — WHERE vs HAVING

**"What is the difference between WHERE and HAVING? Can you use HAVING without GROUP BY?"**

- `WHERE` filters **individual rows** before grouping — it cannot reference aggregate functions
- `HAVING` filters **groups after** `GROUP BY` — it can reference aggregates

```sql
-- WHERE: filter rows before grouping
SELECT dept_id, SUM(sal) AS total_sal
  FROM emp_mstr
 WHERE hire_date > DATE '2020-01-01'   -- row filter
GROUP BY dept_id
HAVING SUM(sal) > 100000;              -- group filter

-- HAVING without GROUP BY: treats entire table as one group
SELECT COUNT(*) AS cnt FROM emp_mstr
 HAVING COUNT(*) > 0;  -- valid, unusual — equivalent to WHERE COUNT(*) > 0 over a dummy group
```

**Why this is asked:** The `WHERE` vs `HAVING` distinction tests whether the candidate understands the logical order of execution: `FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY`. Getting the sequence wrong is a red flag.

> GROUP BY, HAVING, ROLLUP, and subquery patterns: `data-grouping.md`

---

### Question 10 — DECODE vs CASE

**"What does DECODE do? When would you use it instead of CASE, or vice versa?"**

`DECODE` is an Oracle-specific function that performs if-then-else logic inside a `SELECT` clause. It compares an expression to a list of search values and returns the matching result.

```sql
-- DECODE: Oracle-only, compact but limited
SELECT emp_id,
       DECODE(dept_id, 10, 'HR', 20, 'Finance', 30, 'IT', 'Unknown') AS dept_name
  FROM emp_mstr;

-- Equivalent in ANSI SQL (portable across all databases)
SELECT emp_id,
       CASE dept_id
         WHEN 10 THEN 'HR'
         WHEN 20 THEN 'Finance'
         WHEN 30 THEN 'IT'
         ELSE 'Unknown'
       END AS dept_name
  FROM emp_mstr;

-- DECODE with NULL handling (treats NULL = NULL, unlike = operator)
SELECT DECODE(NULL, NULL, 'Match', 'No Match') FROM dual; -- returns 'Match'
-- Standard equality: NULL = NULL returns NULL (not TRUE)
```

**Why this is asked:** `DECODE` is Oracle-specific folklore. A candidate who knows it shows Oracle depth; one who knows `CASE` shows ANSI portability. The best answer uses `CASE` by default and mentions `DECODE` as an Oracle-only shortcut for simple mappings.

> DECODE, NVL, SOUNDEX, hierarchical queries, and 15+ Oracle-specific features: `advance-feat.md`

---

### Quick Reference: Which File Covers What

| Topic                          | Deep-dive file             |
| ------------------------------ | -------------------------- |
| Normalisation (1NF → BCNF)     | `1.intro.md`               |
| All join types (including FULL) | `joins.md`                 |
| B-tree indexes, index damage   | `advanced-sql.md`          |
| Isolation levels, ACID         | `transactions.md`          |
| PK vs UNIQUE, CHECK, FK        | `constraints.md`           |
| Procedures vs functions        | `db-obj.md`                |
| TRUNCATE vs DELETE, ROWNUM     | `advanced-sql.md`, `transactions.md` |
| GROUP BY / HAVING              | `data-grouping.md`         |
| DECODE, NVL, SOUNDEX           | `advance-feat.md`          |
| Control commands, substitution variables | `ctrl-cmds.md`   |
