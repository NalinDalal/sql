# PL/SQL: Procedural Language/SQL

## Introduction
PL/SQL is Oracle's procedural extension to SQL, designed to bridge the gap between database technology and procedural programming. It allows combining SQL with procedural constructs (loops, conditions, variables, error handling) for powerful data manipulation and automation.

- **SQL lacks procedural capabilities** (no loops, conditions, error handling, or permanent storage for variables).
- **PL/SQL is a block-structured language** that enables sending an entire block of SQL statements to the Oracle engine in one go, improving performance and reducing network traffic.
- **PL/SQL is a superset of SQL**—it supports all SQL features plus procedural logic.

## Advantages of PL/SQL
- Supports conditional logic, looping, branching, and error handling.
- Sends an entire block of SQL statements at once, reducing network traffic and improving performance.
- Allows declaration and use of variables and constants.
- Supports transaction control and error management.
- Applications are portable across platforms where Oracle runs.
- Calculations and logic can be performed efficiently without repeated calls to the SQL engine.

## PL/SQL Block Structure
A PL/SQL block has a clear structure:

- **DECLARE**: (optional) Declarations of variables, constants, cursors, etc.
- **BEGIN**: Main executable section (SQL and PL/SQL statements).
- **EXCEPTION**: (optional) Error handling section.
- **END**: Marks the end of the block.

**Example:**
```sql
DECLARE
  -- variable declarations
BEGIN
  -- executable statements
EXCEPTION
  -- error handling
END;
```

## Execution Environment
- PL/SQL blocks are sent to the Oracle engine, which executes procedural statements and SQL as a single unit.
- This reduces the number of calls between application and database, improving efficiency.

## Character Set & Literals
- Supports uppercase/lowercase letters, numerals, and symbols (e.g., +, -, *, /, <, >, =, etc.).
- **Literals**: Numeric (e.g., 25, 6.34), String (e.g., 'Hello'), Character (e.g., 'A'), Boolean (TRUE, FALSE, NULL).

## Data Types
- **number**: Numeric data
- **char**: Fixed-length character data
- **date**: Date and time
- **boolean**: TRUE, FALSE, NULL
- **%TYPE**: Inherit type from table column/variable
- **NOT NULL**: Variable/constant must have a value

## Variables & Constants
- Variables must start with a letter, up to 30 characters, case-insensitive.
- Assign values using `:=` or by fetching from tables.
- Constants use the `CONSTANT` keyword and must be assigned a value at declaration.

## Control Structures
### Conditional Control
- `IF ... THEN ... ELSIF ... ELSE ... END IF;`

**Example:**
```sql
IF balance < min_balance THEN
  -- action
END IF;
```

### Iterative Control
- **Simple Loop:**
  ```sql
  LOOP
    -- statements
    EXIT WHEN condition;
  END LOOP;
  ```
- **WHILE Loop:**
  ```sql
  WHILE condition LOOP
    -- statements
  END LOOP;
  ```
- **FOR Loop:**
  ```sql
  FOR i IN [REVERSE] start..end LOOP
    -- statements
  END LOOP;
  ```

## Exception Handling
- The `EXCEPTION` section handles errors (syntax, logic, validation, etc.).
- Friendly error messages and custom error handling can be implemented.

## Example: Balance Deduction
```sql
DECLARE
  mCUR_BAL NUMBER(11,2);
  mACCT_NO VARCHAR2(7);
  mFINE NUMBER(4) := 100;
  mMIN_BAL CONSTANT NUMBER(7,2) := 5000.00;
BEGIN
  -- Accept account number
  mACCT_NO := '&mACCT_NO';
  -- Retrieve current balance
  SELECT CURBAL INTO mCUR_BAL FROM ACCT_MSTR WHERE ACCT_NO = mACCT_NO;
  -- Deduct fine if balance < min
  IF mCUR_BAL < mMIN_BAL THEN
    UPDATE ACCT_MSTR SET CURBAL = CURBAL - mFINE WHERE ACCT_NO = mACCT_NO;
  END IF;
END;
```

## Comments
- Single-line: `-- comment`
- Multi-line: `/* comment */`

## Useful Packages
- `DBMS_OUTPUT`: Display messages (use `SET SERVEROUTPUT ON`)
- `PUT_LINE`: Print output

## LOB Types
- **BLOB**: Binary data (e.g., images)
- **CLOB**: Character data (e.g., documents)
- **BFILE**: External binary files