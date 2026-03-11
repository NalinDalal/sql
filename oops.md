
# OOPS in Oracle

## Oracle 9i Database Flavours

Oracle 9i supports three main database flavours:
- **Relational**: Traditional Oracle RDBMS.
- **Object-Relational**: Extends relational with object-oriented concepts (abstract datatypes, nested tables, arrays).
- **Object-Oriented**: Database design based on OO analysis and design principles.

Oracle allows you to use all three, but you should be familiar with core relational features and SQL/PLSQL.

---

## Why Use Objects?

- **Object Reuse**: OO code and database objects can be reused, reducing duplication.
- **Standards Adherence**: Using standard datatypes and objects ensures consistency across applications and tables.

Objects combine data and methods. For example, an `address` object might have methods like `Add_Person()`, `Update_Person()`, and `Remove_Person()`.

---

## Object Types in Oracle

### Abstract Datatype

An abstract datatype is a user-defined type with multiple attributes. Example:

```sql
CREATE TYPE ADDRESS_TY AS OBJECT (
	STREET VARCHAR2(50),
	CITY VARCHAR2(25),
	STATE VARCHAR2(25),
	ZIP NUMBER
);
```

### Nested Types

Types can be nested. For example:

```sql
CREATE TYPE PERSON_TY AS OBJECT (
	NAME VARCHAR2(25),
	ADDRESS ADDRESS_TY
);
```

---

## Using Object Types in Tables

You cannot insert data directly into an object type. Instead, create a table using the type:

```sql
CREATE TABLE CUSTOMER (
	CUSTOMER_ID NUMBER,
	PERSON PERSON_TY
);
```

To insert data:

```sql
INSERT INTO CUSTOMER VALUES (
	1,
	PERSON_TY('Sharanam', ADDRESS_TY('Dadar', 'Mumbai', 'Maharashtra', 400016))
);
```

---

## Querying and Manipulating Object Data

- Access nested attributes using dot notation:
	```sql
	SELECT PERSON.ADDRESS.CITY FROM CUSTOMER;
	```
- Update nested attributes:
	```sql
	UPDATE CUSTOMER SET PERSON.ADDRESS.CITY = 'Bombay' WHERE PERSON.ADDRESS.CITY = 'Mumbai';
	```
- Delete using nested attributes:
	```sql
	DELETE FROM CUSTOMER WHERE PERSON.ADDRESS.STREET = 'Dadar';
	```

---

## Object Views

Object views allow you to overlay OO structures on existing relational tables, enabling use of abstract datatypes without rebuilding the application.

Example:
```sql
CREATE OR REPLACE VIEW CUSTOMER_OV (CUSTOMER_ID, PERSON) AS
SELECT CUSTOMER_ID, PERSON_TY(NAME, ADDRESS_TY(STREET, CITY, STATE, ZIP))
FROM CUSTOMER;
```

---

## Nested Tables and Varying Arrays

- **Nested Table**: Table within a table, used for one-to-many relationships.
- **Varying Array (VARRAY)**: Array of objects with a fixed maximum size.

Example VARRAY:
```sql
CREATE TYPE COMPANY_ADDRESS_TY AS VARRAY(3) OF VARCHAR2(1000);
CREATE TABLE COMPANY_INFO (
	COMPANY_NAME VARCHAR2(50),
	ADDRESS COMPANY_ADDRESS_TY
);
```

---

## References (REFs)

REFs are pointers to objects, similar to foreign keys but for object tables.

Example:
```sql
CREATE TYPE DEPT_TY AS OBJECT (
	DNAME VARCHAR2(100),
	ADDRESS VARCHAR2(200)
);

CREATE TABLE DEPT OF DEPT_TY;
CREATE TABLE EMP (
	ENAME VARCHAR2(100),
	ENUMBER NUMBER,
	EDEPT REF DEPT_TY SCOPE IS DEPT
);
```

---

## Summary

- Oracle supports relational, object-relational, and object-oriented models.
- Object types allow encapsulation of data and methods.
- Abstract datatypes, nested tables, and VARRAYs enable complex data modeling.
- Object views and REFs provide flexibility and advanced data relationships.

---