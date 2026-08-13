# OOPs in Oracle

> **Dialect:** Oracle. Same concepts exist in PostgreSQL (object types via `CREATE TYPE ... AS (...)`), but the syntax here is Oracle-flavoured. The "three flavours" taxonomy below is 9i-era marketing; Oracle 23c still supports object-relational features like these, just don't expect the OO flavour to be pitched in new material or interviews.

### Abstract Datatypes

#### Explain It
An abstract datatype is a user-defined object type that groups related attributes — like a mini-schema — created with `CREATE TYPE ... AS OBJECT`. You can't insert data into the type itself; you have to build a table that uses the type as a column, then insert values by calling the type's **constructor** (the type name with `()`). Because types encapsulate data, two reasons to use them are object reuse (one type used across many tables) and standards adherence (every table gets the same shape of address).

#### Prove It
```sql
CREATE TYPE ADDRESS_TY AS OBJECT (
  STREET VARCHAR2(50),
  CITY   VARCHAR2(25),
  STATE  VARCHAR2(25),
  ZIP    NUMBER
);
/
CREATE TABLE ADDR_BOOK (
  PERSON_ID NUMBER,
  ADDR ADDRESS_TY            -- table built on the type
);
INSERT INTO ADDR_BOOK VALUES (
  1,
  ADDRESS_TY('Dadar', 'Mumbai', 'Maharashtra', 400016)
);
```

#### Gotchas / Edge Cases
- The `/` after `CREATE TYPE` is required in SQL*Plus — a type is a PL/SQL-ish statement, not plain SQL, so it needs the slash to execute.
- A bare `CREATE TYPE` name collides with existing objects of any kind: `ORA-00955: name is already used` is what you get when the table/type name is taken (e.g. `DEPT` clashes with the classic `SCOTT` demo synonym — run `CREATE TABLE DEPT_T OF DEPT_TY` with a suffix to dodge it).

---

### Nested Types (Types Within Types)

#### Explain It
A type can itself have an attribute of another user-defined type, creating nesting. This mirrors real-world data: a person *has an* address, so `PERSON_TY` embeds `ADDRESS_TY` as one of its columns. The inner constructor is just called from inside the outer constructor.

#### Prove It
```sql
CREATE TYPE PERSON_TY AS OBJECT (
  NAME    VARCHAR2(25),
  ADDRESS ADDRESS_TY
);
/
CREATE TABLE CUSTOMER (
  CUSTOMER_ID NUMBER,
  PERSON PERSON_TY
);
INSERT INTO CUSTOMER VALUES (
  1,
  PERSON_TY('Sharanam', ADDRESS_TY('Dadar', 'Mumbai', 'Maharashtra', 400016))
);
SELECT C.PERSON.ADDRESS.CITY FROM CUSTOMER C;   -- dot notation walks the nesting
```

#### Gotchas / Edge Cases
- In a SELECT, walk nested attributes with dot notation — but the table **alias is mandatory**: `SELECT C.PERSON.ADDRESS.CITY FROM CUSTOMER C` works, `SELECT PERSON.ADDRESS.CITY FROM CUSTOMER` fails with `ORA-00904`.
- UPDATE/DELETE cannot use a nested attribute in their `WHERE` clause at all (`PERSON.ADDRESS.CITY` is an invalid identifier there), and multi-level `SET` like `SET PERSON.ADDRESS.CITY = '...'` fails too. Options: replace the whole object with a fresh constructor (`SET PERSON = PERSON_TY(...)`), or load it into a PL/SQL variable, change `.ADDRESS.CITY` in memory, and write it back — that path compiles and runs fine.

---

### Object Views

#### Explain It
Object views put object-relational lipstick on plain relational tables: they project an object type over existing columns, so code written against the object model works without rebuilding storage or the application. This is the standard "adopt OO without migrating" trick.

#### Prove It
```sql
CREATE TABLE CUSTOMER_REL (
  CUSTOMER_ID NUMBER,
  NAME    VARCHAR2(25),
  STREET  VARCHAR2(50),
  CITY    VARCHAR2(25),
  STATE   VARCHAR2(25),
  ZIP     NUMBER
);
INSERT INTO CUSTOMER_REL VALUES (2, 'Vaishali', 'Linking Road', 'Mumbai', 'Maharashtra', 400053);

CREATE OR REPLACE VIEW CUSTOMER_OV (CUSTOMER_ID, PERSON) AS
SELECT CUSTOMER_ID, PERSON_TY(NAME, ADDRESS_TY(STREET, CITY, STATE, ZIP))
FROM CUSTOMER_REL;

SELECT CUSTOMER_ID, C.PERSON.ADDRESS.CITY FROM CUSTOMER_OV C;
```

#### Gotchas / Edge Cases
- The constructor call must match the type definition attribute-for-attribute — mistyping an attribute name gives `ORA-00904` pointing at the view's SELECT.
- Same alias rule as above: qualify the object column in queries.

---

### Varying Arrays (VARRAYs)

#### Explain It
A VARRAY is a bounded array — a fixed maximum number of elements of one type, stored as a single column value. Use it when the max cardinality is known (3 phone numbers, 5 addresses). Oracle guarantees ordering of the elements, which nested tables don't.

#### Prove It
```sql
CREATE TYPE COMPANY_ADDRESS_TY AS VARRAY(3) OF VARCHAR2(1000);
/
CREATE TABLE COMPANY_INFO (
  COMPANY_NAME VARCHAR2(50),
  ADDRESS COMPANY_ADDRESS_TY
);
INSERT INTO COMPANY_INFO VALUES ('AstroWorks', COMPANY_ADDRESS_TY('Mumbai', 'Pune'));
SELECT COMPANY_NAME, ADDRESS FROM COMPANY_INFO;
```

#### Gotchas / Edge Cases
- Elements are created in bulk by the constructor; inserting more than the declared limit raises `ORA-22913`.
- VARRAYs are usually inlined into the row, so a huge VARRAY bloats the row — keep them small (the 1000-char example is deliberately oversized).

---

### Nested Tables

#### Explain It
A nested table is an inner table stored as a column — one-to-many without an extra join table. The inner rows live in a system-generated storage table, which you name with the `NESTED TABLE ... STORE AS` clause at `CREATE TABLE` time.

#### Prove It
```sql
CREATE TYPE PHONE_TY AS TABLE OF VARCHAR2(20);
/
CREATE TABLE PHONE_OWNER (
  ID     NUMBER,
  PHONES PHONE_TY
) NESTED TABLE PHONES STORE AS PHONES_TAB;
INSERT INTO PHONE_OWNER VALUES (1, PHONE_TY('9820012345', '9820067890'));
SELECT * FROM TABLE(SELECT PHONES FROM PHONE_OWNER WHERE ID = 1);
```

#### Gotchas / Edge Cases
- Reading a nested table as rows requires `TABLE(...)` around a subquery that returns just that column.
- Dropping the owning table automatically drops the storage table — a cleanup script that also tries to `DROP TABLE PHONES_TAB` afterwards fails with `ORA-00942` on the already-gone table. Catch-and-ignore in cleanup loops.
- Object tables (a table *of* a type) do **not** show up in `USER_TABLES` — they're in `USER_OBJECTS` — so a generic "drop every table" loop silently skips them and the next run trips on `ORA-00955`.

---

### References (REFs)

#### Explain It
A REF is a pointer to an object row in an object table — object-land's foreign key. `SCOPE IS` restricts the REF to one target table, and `REF(D)` with a correlated subquery manufactures the pointer from the target's alias.

#### Prove It
```sql
CREATE TYPE DEPT_TY AS OBJECT (
  DNAME   VARCHAR2(100),
  ADDRESS VARCHAR2(200)
);
/
CREATE TABLE DEPT OF DEPT_TY;
INSERT INTO DEPT VALUES (DEPT_TY('IT', 'Mumbai'));

CREATE TABLE EMP (
  ENAME   VARCHAR2(100),
  ENUMBER NUMBER,
  EDEPT   REF DEPT_TY SCOPE IS DEPT
);
INSERT INTO EMP VALUES ('Ivan', 1, (SELECT REF(D) FROM DEPT D WHERE D.DNAME = 'IT'));
SELECT E.ENAME, E.EDEPT.DNAME FROM EMP E;   -- dereference is done for you
```

#### Gotchas / Edge Cases
- `REF` only works against object tables (`CREATE TABLE X OF type`) — you can't take a REF to a row of an ordinary table.
- Without `SCOPE IS`, the REF is untyped-ish and Oracle can't enforce which table it points at; also, an unqualified scope name resolves against `PUBLIC`, so keep `SCOPE IS DEPT` explicit.
- Traversing a NULL REF (row deleted in the target) makes the attribute access return NULL rather than erroring.

---

## Summary

- Oracle supports relational, object-relational, and object-oriented models; 9i's three-flavours framing is history, but the object-relational toolbox (types, object views, VARRAYs, nested tables, REFs) is still alive in current Oracle.
- Object types encapsulate data and enable reuse; abstract datatypes, nested tables, and VARRAYs model complex shapes without extra join tables.
- Dot notation reads nested attributes, but multi-level predicates in UPDATE/DELETE are unsupported — replace the whole object instead.
- Object views and REFs layer OO behaviour over relational tables without migration.