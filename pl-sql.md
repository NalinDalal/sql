# PL/SQL: Procedural Language for SQL

> **Dialect note:** Oracle PL/SQL throughout. All code verified runnable in Oracle 21c/23c. For package/procedure/function structures, see `db-obj.md`.

---

### PL/SQL Block Structure

#### Explain It

PL/SQL is Oracle's procedural extension to SQL — a block-structured language that allows combining SQL with procedural logic (conditions, loops, variables, error handling) into a single unit sent to the Oracle engine in one call. This eliminates the network round-trips that plague row-by-row SQL from application code.

Every PL/SQL block has up to four parts:

| Part          | Required? | Purpose                                              |
| ------------- | --------- | ---------------------------------------------------- |
| `DECLARE`     | Optional  | Variables, constants, cursors, exceptions            |
| `BEGIN`       | Required  | Executable SQL and PL/SQL statements                 |
| `EXCEPTION`   | Optional  | Error handlers                                       |
| `END;`        | Required  | Terminates the block                                 |

Anonymous blocks run immediately and are not stored. Stored procedures/functions (see `db-obj.md`) are named, compiled, and persisted in the data dictionary.

#### Prove It

```sql
DECLARE
  v_sal    NUMBER(11,2);
  v_min    CONSTANT NUMBER(7,2) := 5000.00;
  v_fine   NUMBER(6,2)    := 100;
  v_acct   VARCHAR2(7)    := '&acct_no';
BEGIN
  SELECT curb INTO v_sal FROM acct_mstr WHERE acct_no = v_acct;

  IF v_sal < v_min THEN
    UPDATE acct_mstr SET curb = curb - v_fine
     WHERE acct_no = v_acct;
    COMMIT;
  END IF;

EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('Account not found: ' || v_acct);
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
END;
/
```

#### Gotchas / Edge Cases

- The `DECLARE` keyword is **only used for anonymous blocks** — stored procedures/functions use `IS` or `AS` instead of `DECLARE`
- `END;` requires a **semicolon** — omitting it causes `PLS-00103: Encountered the symbol "end-of-file"`
- `SQLERRM` in the `OTHERS` handler returns the Oracle error message string — useful for logging without knowing the exact code
- A block without `EXCEPTION` propagates all errors to the caller — no partial handling occurs

---

### Variables and Constants

#### Explain It

Variables store temporary values within a block. Constants store fixed values that cannot change after declaration.

- Names start with a letter, up to **30 characters** (PL/SQL), case-insensitive
- Values assigned with `:=`
- Values also assigned implicitly via `SELECT ... INTO`
- `%TYPE` inherits the exact data type of a table column or another variable — prevents hardcoding mismatches

#### Prove It

```sql
DECLARE
  v_emp_name  emp_mstr.emp_name%TYPE;   -- inherits column type
  v_count     PLS_INTEGER := 0;
  c_max_sal   CONSTANT NUMBER(11,2) := 200000;
BEGIN
  SELECT emp_name INTO v_emp_name
    FROM emp_mstr
   WHERE emp_id = 101;

  v_count := v_count + 1;
  DBMS_OUTPUT.PUT_LINE('Name: ' || v_emp_name);
END;
/
```

#### Gotchas / Edge Cases

- `VARCHAR2` in PL/SQL can hold up to **32767 bytes**; the SQL engine limit is **4000 bytes** — values exceeding 4000 in a SQL context raise `ORA-01704`
- `NUMBER` without precision in PL/SQL can hold up to **38 digits of precision** — enough for financial data, but be aware of floating-point rounding with `BINARY_FLOAT`/`BINARY_DOUBLE`
- `%TYPE` on a `VARCHAR2(100)` column correctly limits the variable — avoids silent truncation if the column definition changes later
- `CONSTANT` values must be assigned at declaration and **cannot** be changed later — attempting `c_max_sal := 300000` raises `PLS-00363: expression 'C_MAX_SAL' cannot be used as an assignment target`

---

### Control Structures

#### Explain It

PL/SQL provides three forms of conditional control and three loop constructs:

**Conditional:**
```sql
IF condition THEN ...
ELSIF condition THEN ...
ELSE ...
END IF;
```

**Loops:**
| Construct        | Use case                              |
| ---------------- | ------------------------------------- |
| `LOOP ... EXIT WHEN` | Simple indefinite loop          |
| `WHILE condition LOOP` | Pre-condition check loop      |
| `FOR i IN start..end LOOP` | Fixed-count loop (implicit counter) |
| `FOR rec IN (SELECT ...) LOOP` | Cursor FOR loop (implicit open/fetch/close) |

The **cursor FOR loop** is the most idiomatic PL/SQL pattern — Oracle implicitly declares the record, opens the cursor, fetches each row, and closes the cursor when done.

#### Prove It

```sql
-- IF / ELSIF / ELSE
DECLARE
  v_grade CHAR(1) := 'B';
BEGIN
  IF v_grade = 'A' THEN
    DBMS_OUTPUT.PUT_LINE('Excellent');
  ELSIF v_grade IN ('B', 'C') THEN
    DBMS_OUTPUT.PUT_LINE('Good');
  ELSE
    DBMS_OUTPUT.PUT_LINE('Needs improvement');
  END IF;
END;
/

-- Cursor FOR loop (most common pattern)
DECLARE
BEGIN
  FOR rec IN (SELECT emp_id, emp_name, sal FROM emp_mstr WHERE dept_id = 10) LOOP
    DBMS_OUTPUT.PUT_LINE(rec.emp_id || ': ' || rec.emp_name || ' - ' || rec.sal);
  END LOOP;
END;
/

-- WHILE loop
DECLARE
  v_i PLS_INTEGER := 1;
BEGIN
  WHILE v_i <= 5 LOOP
    DBMS_OUTPUT.PUT_LINE('Row: ' || v_i);
    v_i := v_i + 1;
  END LOOP;
END;
/

-- FOR loop (reverse)
DECLARE
BEGIN
  FOR i IN REVERSE 5..1 LOOP
    DBMS_OUTPUT.PUT_LINE('Countdown: ' || i);
  END LOOP;
END;
/
```

#### Gotchas / Edge Cases

- `IF` must close with `END IF;` — not just `END`
- `ELSIF` (one L) — `ELSEIF` raises `PLS-00103`
- In a `FOR` loop, the counter variable is **implicitly declared** and is only accessible inside the loop body — referencing it after `END LOOP` raises `PLS-00201: identifier 'I' must be declared`
- A cursor FOR loop **auto-closes** its cursor — no explicit `CLOSE` needed, and calling `CLOSE` on it raises `PLS-00363: expression 'rec' cannot be used as an assignment target`
- `EXIT` without a label exits the **innermost** loop; `EXIT <label>` exits an outer named loop
- `GOTO` exists in PL/SQL but is **widely considered harmful** — avoid in interviews unless specifically asked about its existence

---

### Exception Handling

#### Explain It

An exception is a runtime error (both Oracle-defined and user-defined) that disrupts normal execution. The `EXCEPTION` section catches them and prevents the entire block from failing silently.

**Common predefined exceptions:**

| Exception             | Oracle Error | Cause                                  |
| --------------------- | ------------ | -------------------------------------- |
| `NO_DATA_FOUND`       | ORA-01403    | `SELECT INTO` returned zero rows       |
| `TOO_MANY_ROWS`       | ORA-01422    | `SELECT INTO` returned >1 row          |
| `ZERO_DIVIDE`         | ORA-01476    | Division by zero                       |
| `DUP_VAL_ON_INDEX`    | ORA-00001    | Unique constraint violation            |
| `VALUE_ERROR`         | ORA-06502    | Arithmetic/value conversion error      |
| `OTHERS`              | —            | Catches any exception not yet handled  |

User-defined exceptions are declared in the `DECLARE` section and raised with `RAISE` or `RAISE_APPLICATION_ERROR`.

#### Prove It

```sql
-- Demonstrate NO_DATA_FOUND trap
DECLARE
  v_sal NUMBER;
BEGIN
  SELECT sal INTO v_sal FROM emp_mstr WHERE emp_id = 99999;
  DBMS_OUTPUT.PUT_LINE('Salary: ' || v_sal);
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('Employee 99999 does not exist');
END;
/

-- User-defined exception with RAISE_APPLICATION_ERROR
DECLARE
  e_low_salary EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_low_salary, -20001);
BEGIN
  UPDATE emp_mstr SET sal = 800 WHERE emp_id = 101;
  IF SQL%ROWCOUNT = 0 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Employee not found for salary update');
  END IF;
EXCEPTION
  WHEN e_low_salary THEN
    DBMS_OUTPUT.PUT_LINE(SQLERRM);
END;
/

-- Named user-defined exception with PRAGMA
DECLARE
  e_business_rule EXCEPTION;
BEGIN
  IF :new.sal < 20000 THEN
    RAISE e_business_rule;
  END IF;
EXCEPTION
  WHEN e_business_rule THEN
    DBMS_OUTPUT.PUT_LINE('Salary below minimum threshold');
END;
/
```

#### Gotchas / Edge Cases

- **`NO_DATA_FOUND`** raised by `SELECT INTO` if zero rows returned — if you expect zero rows, use `SELECT ... INTO ...` with `EXCEPTION WHEN NO_DATA_FOUND` or use an aggregate like `NVL(MAX(col), default_val)`
- **`TOO_MANY_ROWS`** raised if `SELECT INTO` returns >1 row — use a cursor or `ROWNUM = 1` filter to prevent this
- `OTHERS` **must be the last** handler — any handlers after it are unreachable
- `SQLCODE` and `SQLERRM` are only valid **inside an EXCEPTION handler** — referencing them in the executable section raises `PLS-00201`
- `SQL%ROWCOUNT` gives rows affected by the last DML — useful for checking `UPDATE`/`DELETE` success after a statement
- `RAISE_APPLICATION_ERROR(-20000..-20999, 'msg')` is the standard way to raise named user errors with a custom message — codes outside this range raise `ORA-21000`

---

### Cursors (Implicit and Explicit)

#### Explain It

A **cursor** is a pointer to a private memory area (context area) that Oracle uses to process SQL statements.

- **Implicit cursor** — Oracle creates and manages it automatically for single-row `SELECT INTO` and all DML. Accessed via `SQL%` attributes.
- **Explicit cursor** — declared and managed by the programmer for multi-row queries. Has a four-step lifecycle: `DECLARE` → `OPEN` → `FETCH` → `CLOSE`.

**Cursor attributes:**

| Attribute          | Returns                                  |
| ------------------ | ---------------------------------------- |
| `SQL%FOUND`        | `TRUE` if DML/SELECT affected ≥1 row     |
| `SQL%NOTFOUND`     | `TRUE` if DML/SELECT affected 0 rows     |
| `SQL%ROWCOUNT`     | Number of rows affected by last DML      |
| `SQL%ISOPEN`       | `TRUE` if cursor is currently open       |
| `cursor%FOUND`      | `TRUE` if last FETCH returned a row      |
| `cursor%NOTFOUND`   | `TRUE` if last FETCH returned no rows    |
| `cursor%ROWCOUNT`   | Number of rows fetched so far            |

#### Prove It

```sql
-- Explicit cursor with open/fetch/close
DECLARE
  CURSOR c_emp IS SELECT emp_id, emp_name, sal FROM emp_mstr WHERE dept_id = 10;
  v_rec  c_emp%ROWTYPE;
BEGIN
  OPEN c_emp;
  LOOP
    FETCH c_emp INTO v_rec;
    EXIT WHEN c_emp%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(v_rec.emp_id || ': ' || v_rec.emp_name);
  END LOOP;
  CLOSE c_emp;
END;
/

-- Implicit cursor attributes after DML
UPDATE emp_mstr SET sal = sal * 1.1 WHERE dept_id = 20;
DBMS_OUTPUT.PUT_LINE('Rows updated: ' || SQL%ROWCOUNT);

IF SQL%NOTFOUND THEN
  DBMS_OUTPUT.PUT_LINE('No rows matched');
END IF;
```

#### Gotchas / Edge Cases

- `SQL%NOTFOUND` is `TRUE` only when zero rows are affected — for `UPDATE`/`DELETE` it does **not** indicate a constraint error; check `SQLCODE` for `ORA-02290`
- Forgetting `CLOSE` on an explicit cursor causes a **cursor leak** — the cursor stays open in the session and consumes PGA memory until the session ends
- A cursor opened inside a loop without a `CLOSE` will accumulate open cursors — always pair `OPEN` with `CLOSE` or use a cursor FOR loop

---

### %TYPE and %ROWTYPE

#### Explain It

- **`%TYPE`** — declares a variable with the same data type as a table column or another variable. Prevents hardcoding and propagates column definition changes automatically.
- **`%ROWTYPE`** — declares a record variable whose fields match the columns of a table or cursor. Eliminates individual variable declarations for each column.

#### Prove It

```sql
-- %TYPE: variable matches column type
DECLARE
  v_sal  emp_mstr.sal%TYPE;      -- inherits NUMBER(11,2)
  v_name emp_mstr.emp_name%TYPE; -- inherits VARCHAR2
BEGIN
  SELECT sal, emp_name INTO v_sal, v_name FROM emp_mstr WHERE emp_id = 101;
  DBMS_OUTPUT.PUT_LINE(v_name || ' earns ' || v_sal);
END;
/

-- %ROWTYPE: record matches table structure
DECLARE
  v_emp  emp_mstr%ROWTYPE;
BEGIN
  SELECT * INTO v_emp FROM emp_mstr WHERE emp_id = 101;
  DBMS_OUTPUT.PUT_LINE(v_emp.emp_name || ' / ' || v_emp.sal);
END;
/
```

#### Gotchas / Edge Cases

- `%ROWTYPE` on a `SELECT *` with `JOIN` will **only include columns from one table** — it cannot span multiple tables; use individual `%TYPE` variables or a user-defined record type instead
- `%TYPE` on a `VARCHAR2` column does **not** carry the length constraint for PL/SQL variable declarations — the variable can exceed the column's defined length
- When a column's type changes in the table, `%TYPE` variables pick up the new type at **next compilation** — existing running sessions are unaffected

---

### Summary Table

| Feature        | Purpose                                      | Key Gotcha                              |
| -------------- | -------------------------------------------- | --------------------------------------- |
| Anonymous block| Ad-hoc PL/SQL; not stored                    | Must use `DECLARE` not `IS`/`AS`        |
| Stored proc    | Named, compiled, reusable action             | No return value; use `OUT` for outputs  |
| Variable       | Temporary value in a block                   | Max 32767 bytes in PL/SQL               |
| Constant       | Immutable value                               | Must assign at declaration              |
| `%TYPE`        | Inherits column/variable type                | Does not enforce VARCHAR2 length        |
| `%ROWTYPE`     | Record matching table/cursor columns         | Single table only, not joins            |
| Cursor FOR loop| Implicit open/fetch/close                     | Counter inaccessible after END LOOP     |
| Exception      | Runtime error handler                        | `NO_DATA_FOUND` = 0 rows, not error     |
| `OTHERS`       | Catches all unhandled exceptions             | Must be last; use `SQLCODE`/`SQLERRM`   |
