# PL/SQL Database Objects

> **Dialect note:** Oracle PL/SQL throughout. All code verified runnable in Oracle 21c/23c.

---

### Stored Procedures and Functions

#### Explain It

A **stored procedure or function** is a named PL/SQL code block compiled and stored in Oracle's system tables. Parameters make them dynamic. They exist in three parts:

- **Declarative Part** — declarations of cursors, variables, constants, exceptions; local to the block
- **Executable Part** — SQL and PL/SQL statements
- **Exception Handling Part** — catches errors; cannot transfer execution back to the executable part

Before storing, the Oracle engine **parses and compiles** the block. When invoked, it is loaded into the **System Global Area (SGA)**, enabling fast shared execution across users.

**Execution steps by Oracle engine:**
1. Verify user has `EXECUTE` privilege
2. Verify object status is `VALID`
3. Execute

**Key advantage over ad-hoc SQL:** logic is compiled once, stored in the database, and reusable — reducing network round-trips and enforcing business rules at the data layer.

#### Prove It

```sql
-- Create a procedure to apply a salary hike
CREATE OR REPLACE PROCEDURE apply_salary_hike(
  p_emp_id    IN  NUMBER,
  p_pct       IN  NUMBER
) AS
BEGIN
  UPDATE emp_mstr
     SET sal = sal * (1 + p_pct / 100)
   WHERE emp_id = p_emp_id;
  COMMIT;
END apply_salary_hike;
/

-- Invoke it
EXEC apply_salary_hike(101, 10);

-- Confirm it compiled and is valid
SELECT object_name, object_type, status
  FROM user_objects
 WHERE object_name = 'APPLY_SALARY_HIKE';

-- View any compilation errors
SELECT * FROM user_errors WHERE name = 'APPLY_SALARY_HIKE';
```

#### Gotchas / Edge Cases

- `CREATE OR REPLACE` **replaces** the object atomically — dependent objects are invalidated but auto-recompiled on next call
- `OUT` parameters cannot be **read inside** the procedure before being assigned; they return values only
- DML inside a procedure requires the caller (or the procedure itself) to `COMMIT` — Oracle does not auto-commit
- Calling a procedure that does DML from a **SQL statement** (e.g., inside a trigger or function called from SQL) raises `ORA-14551: cannot perform a DML operation inside a query`

---

### Procedures vs Functions

#### Explain It

The core difference: a **procedure** performs an action; a **function** computes and returns a value.

| Aspect           | Procedure                    | Function                          |
| ---------------- | ---------------------------- | --------------------------------- |
| Return value     | Optional (via `OUT` params)  | **Mandatory** — exactly one value |
| Multiple outputs | Yes (multiple `OUT` params)  | No                                |
| Callable from SQL| No (only via `EXEC`/`CALL`)  | Yes (in `SELECT`, `WHERE`, etc.)  |
| DML restriction  | None (called via `EXEC`)     | Cannot DML unless autonomous tx   |

Functions accept only `IN` parameters. Procedures accept `IN`, `OUT`, and `IN OUT`.

#### Prove It

```sql
-- Function: compute annual salary (callable from SQL)
CREATE OR REPLACE FUNCTION annual_salary(
  p_emp_id IN NUMBER
) RETURN NUMBER AS
  v_sal NUMBER;
BEGIN
  SELECT sal INTO v_sal FROM emp_mstr WHERE emp_id = p_emp_id;
  RETURN v_sal * 12;
END;
/

-- Call from a SELECT statement
SELECT emp_id, annual_salary(emp_id) AS annual_sal FROM emp_mstr;

-- Procedure: apply hike (not callable from SQL directly)
CREATE OR REPLACE PROCEDURE apply_hike(
  p_emp_id IN NUMBER,
  p_pct    IN NUMBER
) AS
BEGIN
  UPDATE emp_mstr SET sal = sal * (1 + p_pct / 100)
   WHERE emp_id = p_emp_id;
END apply_hike;
/

-- Invoke via EXEC (not SELECT)
EXEC apply_hike(101, 10);
```

#### Gotchas / Edge Cases

- A **function called from SQL** cannot contain DML unless declared `PRAGMA AUTONOMOUS_TRANSACTION` — Oracle raises `ORA-14551`
- `IN OUT` parameters accept and return a value in the same variable; they cannot be used in functions (functions have no `IN OUT`)
- `RETURN` in a procedure is for **early exit only** — it does not return a value; use it without an expression

---

### Oracle Packages

#### Explain It

A **package** is a schema-level object that groups related procedures, functions, variables, constants, cursors, and exceptions into a single logical unit. Packages have two parts:

- **Package Specification** — declares the public interface (types, subprogram signatures, constants). Users need `EXECUTE` on the package to call anything declared here.
- **Package Body** — fully implements the declared subprograms; may also contain private declarations inaccessible outside the package.

When a package is first called in a session, Oracle loads it into the **SGA**. Subsequent calls within the same session are fast (no disk I/O). Package-level variables and cursors **persist for the entire session**.

#### Prove It

```sql
-- Package specification
CREATE OR REPLACE PACKAGE emp_mgmt AS
  PROCEDURE hire_emp(p_emp_id IN NUMBER, p_name IN VARCHAR2, p_sal IN NUMBER);
  FUNCTION  get_salary(p_emp_id IN NUMBER) RETURN NUMBER;
  v_min_salary CONSTANT NUMBER := 20000;
END emp_mgmt;
/

-- Package body
CREATE OR REPLACE PACKAGE BODY emp_mgmt AS
  PROCEDURE hire_emp(p_emp_id IN NUMBER, p_name IN VARCHAR2, p_sal IN NUMBER) IS
  BEGIN
    INSERT INTO emp_mstr(emp_id, emp_name, sal)
    VALUES (p_emp_id, p_name, p_sal);
    COMMIT;
  END hire_emp;

  FUNCTION get_salary(p_emp_id IN NUMBER) RETURN NUMBER IS
    v_sal NUMBER;
  BEGIN
    SELECT sal INTO v_sal FROM emp_mstr WHERE emp_id = p_emp_id;
    RETURN v_sal;
  END get_salary;
END emp_mgmt;
/

-- Invoke using dot notation
EXEC emp_mgmt.hire_emp(201, 'NewHire', 25000);
SELECT emp_mgmt.get_salary(201) AS salary FROM dual;

-- Recompile a package
ALTER PACKAGE emp_mgmt COMPILE BODY;

-- Recompile all invalid objects
EXEC DBMS_UTILITY.COMPILE_ALL;
```

#### Gotchas / Edge Cases

- A **package cannot be called directly** — only its individual procedures/functions are invoked via dot notation (`pkg_name.subprogram_name`)
- **Recompiling a package invalidates** all dependent objects; Oracle auto-recompiles at runtime but concurrent sessions may see `ORA-04068` until it stabilises
- Package variables declared in the **specification** are session-persistent but **reset** if the package is recompiled mid-session (`ORA-04068: state of package has been discarded`)
- Objects with `PRIVATE` declarations (only in the body) cannot be accessed from outside the package — they are invisible to callers

---

### Overloading Procedures and Functions

#### Explain It

**Overloading** allows multiple subprograms with the **same name but different parameter signatures** within a package or PL/SQL block. Oracle resolves the correct version at runtime based on the number and data types of arguments passed.

Overloading eliminates long `IF/CASE` chains for type dispatch and lets callers use a single logical name regardless of input type.

**Restrictions:**
- At least one parameter's data type must differ — overloading by numeric subtypes (`INTEGER` vs `NUMBER`) or `CHAR` vs `VARCHAR2` is **not allowed** (Oracle treats them as compatible)
- Parameter mode alone (`IN` vs `IN OUT`) is not enough to distinguish overloads — Oracle cannot resolve these at call time

#### Prove It

```sql
CREATE OR REPLACE PACKAGE calc_pkg AS
  -- Overloaded: takes a DATE, returns VARCHAR2
  FUNCTION describe_date(d IN DATE) RETURN VARCHAR2;
  -- Overloaded: takes a NUMBER, returns VARCHAR2
  FUNCTION describe_date(n IN NUMBER) RETURN VARCHAR2;
END calc_pkg;
/

CREATE OR REPLACE PACKAGE BODY calc_pkg AS
  FUNCTION describe_date(d IN DATE) RETURN VARCHAR2 IS
  BEGIN
    RETURN 'Date: ' || TO_CHAR(d, 'YYYY-MM-DD');
  END;

  FUNCTION describe_date(n IN NUMBER) RETURN VARCHAR2 IS
  BEGIN
    RETURN 'Number: ' || TO_CHAR(n);
  END;
END calc_pkg;
/

-- Oracle picks the right version based on argument type
SELECT calc_pkg.describe_date(SYSDATE)     FROM dual; -- calls DATE version
SELECT calc_pkg.describe_date(42)           FROM dual; -- calls NUMBER version
```

#### Gotchas / Edge Cases

- Overloading is **only allowed inside a PL/SQL block or a package** — two standalone procedures cannot share a name
- `CHAR` vs `VARCHAR2` overloading **does not work** — Oracle's type compatibility rules treat them as the same family for resolution
- `NUMBER(3)` vs `NUMBER(6)` overloading **does not work** — numeric precision differences are ignored at resolution time
- If Oracle cannot find a unique match, it raises `PLS-00307: too many declarations of '<name>' match this call`

---

### Summary Table

| Object        | Purpose                                      | Key Point                                      |
| ------------- | -------------------------------------------- | ---------------------------------------------- |
| Procedure     | Named PL/SQL block, no mandatory return      | Uses `OUT`/`IN OUT` for multiple return values  |
| Function      | Named PL/SQL block                           | Must return exactly one value; callable from SQL|
| Package Spec  | Declares public interface                    | Global to the package; grant EXECUTE here       |
| Package Body  | Implements spec + private objects            | Private objects inaccessible outside            |
| Overloading   | Same name, different param types             | Only inside packages or PL/SQL blocks           |
