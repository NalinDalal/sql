## PL/SQL Security

### 1. Concurrency Control
- Oracle tables are global resources accessed by multiple users.
- Concurrency control ensures data integrity when multiple users access/update data.

### 2. Locking
- Oracle uses locks to manage concurrent access.
- Implicit Locking: Automatic by Oracle engine.
- Explicit Locking: User-defined (e.g., SELECT ... FOR UPDATE, LOCK TABLE).

### 3. Types of Locks
- Shared Lock: For read operations (SELECT), allows multiple users.
- Exclusive Lock: For write operations (INSERT, UPDATE, DELETE), only one user.

### 4. Levels of Locks
- Row level
- Page level
- Table level
- Oracle does not provide field-level locks.

### 5. Locking Rules
- Data being changed cannot be read.
- Writers wait for other writers.

### 6. Deadlock
- Occurs when two transactions wait for each other’s lock.
- Oracle detects and resolves deadlocks automatically.

### 7. Exception Handling
- Oracle raises exceptions for errors (e.g., resource busy, deadlock).
- Exception handlers: Named (predefined) and user-defined.

### 8. Key SQL Syntax
- SELECT ... FOR UPDATE [NOWAIT]
- LOCK TABLE <table> IN [EXCLUSIVE|SHARE] MODE [NOWAIT]
- Exception handling: EXCEPTION WHEN <ExceptionName> THEN <Action>

---

### 9. Detailed Locking Concepts
- Implicit Locking: Oracle automatically applies locks based on operation type.
- Explicit Locking: User can override default by using SELECT ... FOR UPDATE or LOCK TABLE.
- Shared Lock: For SELECT/read operations; multiple users can read simultaneously.
- Exclusive Lock: For INSERT/UPDATE/DELETE; only one user can write.

#### Lock Levels
- Row Level: Locks individual rows (WHERE clause matches one row).
- Page Level: Locks a set of data (WHERE clause matches a set).
- Table Level: Locks entire table (no WHERE clause).

#### Locking Rules
- Data being changed cannot be read.
- Writers wait for other writers.

---

### 10. Explicit Locking Syntax & Examples
- SELECT ... FOR UPDATE [NOWAIT]: Acquires exclusive lock for update.
- LOCK TABLE <table> IN [EXCLUSIVE|SHARE] MODE [NOWAIT]: Manually locks table.

Example:
```sql
Client A: SELECT * FROM ACCT_MSTR WHERE ACCT_NO = 'SB9' FOR UPDATE;
Client B: SELECT * FROM ACCT_MSTR WHERE ACCT_NO = 'SB9' FOR UPDATE;
-- Client B waits or gets 'resource busy' if NOWAIT is used
```

---

### 11. Observations & Inferences from Multi-User Operations
- If a record is locked for update, other users cannot acquire lock until commit/rollback.
- SELECT ... FOR UPDATE NOWAIT returns error if resource is busy: ORA-00054.
- Only committed changes are visible to other users.
- Uncommitted changes are visible only to the user who made them.

---

### 12. Deadlock
- Occurs when two transactions wait for each other’s lock.
- Oracle detects and resolves deadlocks automatically by aborting one transaction.

Example:
```sql
Transaction 1:
UPDATE ACCT_MSTR SET CURBAL = 500 WHERE ACCT_NO='SB1';
UPDATE ACCT_MSTR SET CURBAL = 2500 WHERE ACCT_NO='CA2';

Transaction 2:
UPDATE ACCT_MSTR SET CURBAL = -5000 WHERE ACCT_NO='CA2';
UPDATE ACCT_MSTR SET CURBAL = 3500 WHERE ACCT_NO='SB1';
```

---

### 13. Exception Handling in PL/SQL
- Oracle raises exceptions for errors (resource busy, deadlock, etc.).
- Named Exception Handlers: Predefined (e.g., NO_DATA_FOUND, TOO_MANY_ROWS, LOGIN_DENIED).
- User-Defined Exception Handlers: Use PRAGMA EXCEPTION_INIT to bind error codes.
- Business Rule Validation: Use RAISE to trap business rule violations.

Syntax:
```sql
DECLARE
  <ExceptionName> EXCEPTION;
  PRAGMA EXCEPTION_INIT(<ExceptionName>, <ErrorCodeNo>);
BEGIN
  -- SQL statements
EXCEPTION
  WHEN <ExceptionName> THEN <Action>;
END;
```

Example:
```sql
DECLARE
  RESOURCE_BUSY EXCEPTION;
  PRAGMA EXCEPTION_INIT(RESOURCE_BUSY, -00054);
BEGIN
  SELECT CURBAL INTO ACCT_BALANCE FROM ACCT_MSTR WHERE ACCT_NO = 'SB1' FOR UPDATE NOWAIT;
EXCEPTION
  WHEN RESOURCE_BUSY THEN
    DBMS_OUTPUT.PUT_LINE('The row is in use');
END;
```

---

### 14. Key Error Codes
- ORA-00054: resource busy and acquire with NOWAIT specified
- ORA-06512: line number in PL/SQL block

---

### 15. Business Rule Exception Handling
- Use RAISE to trap business rule violations (e.g., withdrawal exceeds balance).

Example:
```sql
DECLARE
  MORE_THAN_BAL EXCEPTION;
BEGIN
  -- If withdrawal > balance, RAISE MORE_THAN_BAL;
EXCEPTION
  WHEN MORE_THAN_BAL THEN
    DBMS_OUTPUT.PUT_LINE('Withdrawal exceeds balance');
END;
```

---

### 16. Summary Table: Locking & Exception Handling

| Concept                | Syntax/Example                                      |
|------------------------|-----------------------------------------------------|
| Implicit Locking       | Automatic by Oracle engine                          |
| Explicit Locking       | SELECT ... FOR UPDATE, LOCK TABLE                   |
| Shared Lock            | SELECT (read)                                       |
| Exclusive Lock         | INSERT/UPDATE/DELETE (write)                        |
| Deadlock               | Two transactions wait for each other                |
| Exception Handling     | EXCEPTION WHEN <ExceptionName> THEN <Action>        |
| Named Exception        | NO_DATA_FOUND, TOO_MANY_ROWS, LOGIN_DENIED          |
| User-Defined Exception | PRAGMA EXCEPTION_INIT, RAISE                        |
| Business Rule          | RAISE <ExceptionName>                               |