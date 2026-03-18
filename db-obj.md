# PL/SQL DATABASE OBJECTS

A Procedure or Function is a logically grouped set of SQL and PL/SQL statements that perform a specific task.
A stored procedure or function is a **named PL/SQL code block** that has been compiled and stored in one of Oracle's system tables.
Parameters can be passed to make them dynamic.

## Structure of Procedures / Functions

Made up of three parts:

- **Declarative Part** — declarations of cursors, constants, variables, exceptions; local to the block, become invalid once it exits
- **Executable Part** — SQL and PL/SQL statements that assign values, control execution, manipulate data, and return results
- **Exception Handling Part** — handles exceptions raised in the executable part; **cannot transfer execution back** to the executable part

---

## Where Are They Stored?

Stored in the Oracle database. Before storing, the Oracle engine **parses and compiles** them.

To view compilation errors:

```sql
SELECT * FROM USER_ERRORS;
```

To check status of a procedure or function:

```sql
SELECT Object_Name, Object_Type, Status FROM User_Objects
  WHERE Object_Type = 'PROCEDURE';
-- or
  WHERE Object_Type = 'FUNCTION';
```

When invoked, Oracle loads the compiled block into the **System Global Area (SGA)** — allowing fast execution and shared access by multiple users (if granted permission).

### Execution Steps by Oracle Engine

1. Verify user access (EXECUTE privilege)
2. Verify validity (status must be VALID)
3. Execute

---

## Advantages of Procedures / Functions

| Advantage        | Description                                                    |
| ---------------- | -------------------------------------------------------------- |
| **Security**     | Grant execute on procedure instead of direct table access      |
| **Performance**  | Less network traffic, no compilation at runtime, less disk I/O |
| **Memory**       | Shared memory — only one copy loaded for multiple users        |
| **Productivity** | Avoids redundant coding                                        |
| **Integrity**    | Tested once, maintained by the Oracle engine                   |

---

## Procedures vs Functions

|                    | Procedure                            | Function                          |
| ------------------ | ------------------------------------ | --------------------------------- |
| Return value       | Not mandatory (uses OUT params)      | **Must** return exactly one value |
| Multiple outputs   | Yes, via multiple OUT parameters     | No                                |
| OUT variable scope | Global — accessible by calling block | N/A                               |

---

## Creating a Stored Procedure

```sql
CREATE OR REPLACE PROCEDURE [Schema.] <ProcedureName>
  (<Argument> {IN | OUT | IN OUT} <DataType>, ...) {IS | AS}
  <Variable declarations>;
  <Constant declarations>;
BEGIN
  <PL/SQL subprogram body>;
EXCEPTION
  <Exception PL/SQL block>;
END;
```

- `IN` — accepts value from caller
- `OUT` — returns value to caller
- `IN OUT` — accepts and returns value

---

## Creating a Function

```sql
CREATE OR REPLACE FUNCTION [Schema.] <FunctionName>
  (<Argument> IN <DataType>, ...)
  RETURN <DataType> {IS | AS}
  <Variable declarations>;
  <Constant declarations>;
BEGIN
  <PL/SQL subprogram body>;
EXCEPTION
  <Exception PL/SQL block>;
END;
```

- Functions only accept `IN` parameters
- `RETURN <DataType>` is **mandatory**

---

## Deleting a Procedure / Function

```sql
DROP PROCEDURE <ProcedureName>;
DROP FUNCTION <FunctionName>;
```

---

## Oracle Packages

A package is an Oracle object that **holds other objects** within it — procedures, functions, variables, constants, cursors, and exceptions.

- Created using SQL\*Plus
- Compiled and stored in Oracle's system tables
- All users with EXECUTE permission can use it
- **The package itself cannot be called, passed parameters to, or nested**

### Components of a Package

1. **Package Specification** — declares types, variables, constants, exceptions, cursors, and subprograms (public interface)
2. **Package Body** — fully defines the declared objects; can also contain private declarations not accessible outside

### Why Use Packages?

1. Organizes applications into efficient, well-defined modules
2. Allows efficient privilege granting
3. Public variables and cursors persist for the **entire session**
4. Enables **overloading** of procedures and functions
5. Loads multiple objects into memory at once — reduces I/O
6. Promotes code reuse and reduces redundant coding

---

## Package Specification Syntax

```sql
CREATE OR REPLACE PACKAGE <PackageName> AS
  PROCEDURE <ProcName>(<args>);
  FUNCTION <FuncName>(<args>) RETURN <DataType>;
  <variable> <DataType>;
END <PackageName>;
```

## Package Body Syntax

```sql
CREATE OR REPLACE PACKAGE BODY <PackageName> AS
  PROCEDURE <ProcName>(<args>) IS
  BEGIN
    ...
  END;

  FUNCTION <FuncName>(<args>) RETURN <DataType> IS
  BEGIN
    ...
  END;
END <PackageName>;
```

---

## Invoking a Package

Oracle performs 3 steps:

1. Verify user access (EXECUTE privilege)
2. Verify subprogram validity (auto-recompiles if invalid)
3. Execute

Use **dot notation** to reference package objects:

```sql
PackageName.TypeName
PackageName.ObjectName
PackageName.SubprogramName
```

Example:

```sql
EXECUTE TRANSACTION_MGMT.PERFORM_TRANS('SBI1', 'D', 5000);
-- or
CALL TRANSACTION_MGMT.CANCEL_FD('F1');
```

---

## Altering / Recompiling a Package

```sql
ALTER PACKAGE <PackageName> COMPILE PACKAGE;  -- recompile spec
ALTER PACKAGE <PackageName> COMPILE BODY;     -- recompile body only
```

Recompile all packages:

```sql
EXECUTE DBMS_UTILITY.COMPILE_ALL;
```

> Recompiling a package **invalidates all dependent objects**. Oracle auto-recompiles them at runtime if needed.

---

## Public vs Private Package Objects

|                    | Public                 | Private           |
| ------------------ | ---------------------- | ----------------- |
| Declared in        | Package Specification  | Package Body only |
| Accessible outside | Yes (via dot notation) | No                |

---

## Variable / Cursor / Constant Lifetime

- Declared in **standalone procedure** → exists only during that procedure call
- Declared in **package specification or body** → persists for the **entire user session**, lost when session ends or package is recompiled

---

## Package State

- **Valid** — no referenced objects have been dropped, altered, or replaced since last compile
- **Invalid** — a referenced object was changed; Oracle also invalidates any object that references this package

---

## Overloading Procedures and Functions

Multiple procedures/functions with the **same name but different parameters** defined in a package or PL/SQL block.

```sql
CREATE OR REPLACE PACKAGE CHECK_FUNC IS
  FUNCTION VALUE_OK(DATE_IN IN DATE) RETURN VARCHAR2;
  FUNCTION VALUE_OK(NUMBER_IN IN NUMBER) RETURN VARCHAR2;
END;
```

Oracle picks the right one based on the data type of the passed argument.

### Benefits of Overloading

1. Eliminates multiple IF/CASE constructs to check parameter types
2. Transfers burden of knowledge from developer to the software
3. Programmer needs to remember only **one name** regardless of parameter type

### Where Can You Overload?

- Inside the **declaration section** of a PL/SQL block
- Inside a **package**
- **NOT** in standalone programs (two standalone procedures cannot share a name)

### Restrictions on Overloading

1. At least **one parameter's data type must differ** — overloading by different numeric subtypes (e.g., INTEGER vs NUMBER) or CHAR vs VARCHAR2 is **not allowed**
2. Parameter lists must differ by **more than just name or parameter mode** (IN vs IN OUT is not enough — Oracle can't distinguish these at call time)

---

## Summary Table

| Object       | Purpose                                 | Key Point                             |
| ------------ | --------------------------------------- | ------------------------------------- |
| Procedure    | Named PL/SQL block, no mandatory return | Uses OUT for multiple return values   |
| Function     | Named PL/SQL block                      | Must return exactly one value         |
| Package Spec | Declares public interface               | Global to the package                 |
| Package Body | Implements spec + private objects       | Private objects inaccessible outside  |
| Overloading  | Same name, different param types        | Only inside packages or PL/SQL blocks |
