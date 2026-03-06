page 137 - 155
Chap 8 INTERACTIVE-SQL PART II

# DATA CONSTRAINTS
to check DATA for integrity prior storage.Once it is part the db engine checks
the inputs; if passed then gets entered into table else rejected

Business rules, which are enforced on data being stored in a table, are called Constraints. Constraints,
super control the data being entered into a table for permanent storage

ALTER TABLE and CREATE TABLE can be used for such

Types:
- I/O Constraint
- Business Rule

1. I/O Constraints
    - Primary Key
    - Foreign Key

2. Business Constraints
    - Column Level
    - NULL Value
    - CHECK

## Primary Constraints
PRIMARY KEY
- NOT NULL
- UNIQUE

**Simple Key:** single column primary key
**Composite Key:** multicolumn primary key

*PRIMARY KEY* is basically used to identify rows uniquely.if it doesn't works,
define *COMPOSITE KEY*.

### Features Primary Key
1. Primary key is a column or a set of columns that uniquely identifies a row. Its main purpose is the `Record Uniqueness`
2. Primary key will not allow duplicate values 
3. Primary key will also not allow null values
4. Primary key is not compulsory but it is recommended
5. Primary key helps to identify one record from another record and also helps in relating tables with one another
6. Primary key cannot be LONG or LONG RAW data type
7. Only one Primary key is allowed per table
8. Unique Index is created automatically if there is a Primary key
9. One table can combine upto 16 columns in a Composite Primary key

Syntax(at column level):
```sql
<ColumnName> <Datatype>(<Size>) PRIMARY KEY
```

ex: Create a table CUST_MSTR such that the contents of the column CUST_NO is unique and not null.
```sql
DROP TABLE CUST_ MSTR;
CREATE TABLE CUST_MSTR (
"CUST_NO" VARCHAR2(10) PRIMARY KEY,
"FNAME" VARCHAR2(25), "MNAME" VARCHAR2(25),
"LNAME" VARCHAR2(25), "DOB INC" DATE,
"OCCUP" VARCHAR2(25), "PHOTOGRAPH" VARCHAR2(25),
"SIGNATURE" VARCHAR2(25), "PANCOPY" VARCHAR2(1),
"FORM60" VARCHAR2(1));
```

<br>

Syntax(at Table Level):
```sql
PRIMARY KEY (<ColumnName>, <ColumnName>)
```
ex: table FD_MSTR where there is a composite primary key mapped to the columns FD_SER_NO and CORP_CUST_NO.
```sql
DROP TABLE FD MSTR;
CREATE TABLE "DBA_BANKSYS"."FD_MSTR"(
"ED SER NO" VARCHAR2(10), "SF_NO" VARCHAR2(10),
"BRANCH NO" VARCHAR2(10), "INTRO_CUST_NO" VARCHAR2(10),
"INTRO. ACCT_NO" VARCHAR2(10), "INTRO_SIGN" VARCHAR2(1),
"ACCT NO" VARCHAR2(10), "TITLE" VARCHAR2(30),
"CORP CUST NO" VARCHAR2(10), "CORP_CNST_TYPE" VARCHAR(4),
"VERI EMP NO" VARCHAR2(10), "VERL SIGN" VARCHAR2(1),
"MANAGER SIGN" VARCHAR2(1), PRIMARY KEY(FD_SER NO, CORP_CUST_NO));
```

## Foreign Key
A foreign key is a column (or a group of columns) whose values are derived from the primary key or unique key of some other table.
The table in which the foreign key is defined is called a Foreign table or Detail table. 
The table that defines the primary or unique key and is referenced by the foreign key is called the Primary table or Master table.

Features of Foreign Keys
1. Foreign key is a column(s) that references a column(s) of a table and it can be the same table also
2. Parent that is being referenced has to be unique or Primary key
3. Child may have duplicates and nulls but unless it is specified
4. Foreign key constraint can be specified on child but not on parent
5. Parent record can be delete provided no child record exist
6. Master table cannot be updated if child record exist

**Note:** A foreign key must have a corresponding primary key or unique key value in the master table.
**Note:** displays an error message when a record in the master table is deleted and corresponding records exists in a detail table and prevents the delete operation from going through.

### Principles:
- Rejects an INSERT or UPDATE of a value, if a corresponding value does not currently exist in the master key table
- If the ON DELETE CASCADE option is set, a DELETE operation in the master table will trigger a DELETE operation for corresponding records in all detail tables
- Ifthe ON DELETE SET NULL option is set, a DELETE operation in the master table will set the value held by the foreign key of the detail tables to null
- Rejects a DELETE from the Master table if corresponding records in the DETAIL table exist
- Must reference a PRIMARY KEY or UNIQUE column(s) in primary table
- Requires that the FOREIGN KEY column(s) and the CONSTRAINT column(s) have matching data types
- Can reference the same table named in the CREATE TABLE statement

Syntax(at Column Level):
```sql
<ColumnName> <DataType>(<Size>)
REFERENCES <TableName> [(<ColumnName>)]
[ON DELETE CASCADE]
```
ex: `DROP TABLE EMP_MSTR;
CREATE TABLE "DBA_BANKSYS"."EMP_MSTR"(
"EMP. NO" VARCHAR2(10) PRIMARY KEY,
"BRANCH. NO" VARCHAR2(10) REFERENCES BRANCH _ MSTR,
"FNAME" VARCHAR2(25), "MNAME" VARCHAR2(25),
"LNAME" VARCHAR2(25), "DEPT" VARCHAR2(30),
"DESIG" VARCHAR2(30));`

Syntax(at Table Level):
```sql
FOREIGN KEY (<ColumnName> [,<ColumnName>] )
    REFERENCES <TableName> [(<ColumnName>, <ColumnName >)
```

ex: `DROP TABLE ACCT FD CUST_DTLS;
CREATE TABLE "DBA_BANKSYS"."ACCT_FD CUST_DTLS"(
"ACCT FD NO" VARCHAR2(10), "CUST_NO" VARCHAR2(10),
FOREIGN KEY (CUST_NO) REFERENCES CUST_MSTR(CUST_NO));`

Syntax(With ON DELETE CASCADE)
```sql
CREATE TABLE "DBA_BANKSYS"."FD_DTLS"(
"FD_SER_NO" VARCHAR2(10), "FD_NO" VARCHAR2(10),
"TYPE" VARCHAR2(1), "PAYTO_ACCTNO" VARCHAR2(10),
"PERIOD" NUMBER(S), "OPNDT" DATE,
"DUEDT" DATE, "AMT" NUMBER(8,2),
"DUEAMT" NUMBER(8,2), "INTRATE" NUMBER(3),
"STATUS" VARCHAR2(1) DEFAULT 'A', "AUTO RENEWAL" VARCHAR2(1) ,
CONSTRAINT f FDSerNoKey
FOREIGN KEY (FD_SER_NO) REFERENCES FD _MSTR(FD SER NO)
ON DELETE CASCADE), .
```

Syntax(with ON DELETE SET NULL):
```sql
DROP TABLE FD DTLS:
CREATE TABLE "DBA_BANKSYS"."FD_DTLS"(
"FD SER NO" VARCHAR2(10), "FD_NO" VARCHAR2(10), "TYPE" VARCHAR2(1),
"PAYTO_ACCTNO" VARCHAR2(10), "PERIOD" NUMBER(S), "OPNDT" DATE,
"DUEDT" DATE, "AMT" NUMBER(8,2), "DUEAMT" NUMBER(8,2), "INTRATE" NUMBER(3),
"STATUS" VARCHAR2(1) DEFAULT 'A', "AUTO_RENEWAL" VARCHAR2(1) ,
CONSTRAINT f FDSerNoKey
FOREIGN KEY (FD_SER_NO) REFERENCES FD _MSTR(FD_SER_NO)
ON DELETE SET NULL);
```

### Assigning User Defined Names To Constraints
```sql
CONSTRAINT <Constraint Name> <Constraint Definition>
```
Ex: 
```sql
CREATE TABLE CUST_MSTR (
"CUST_NO" VARCHAR2(10) CONSTRAINT p CUSTKey PRIMARY KEY,
"FNAME" VARCHAR2(25), "MNAME" VARCHAR2(25),
"LNAME" VARCHAR2(25), "DOB_INC" DATE,
"OCCUP" VARCHAR2(25), "PHOTOGRAPH" VARCHAR2(25),
"SIGNATURE" VARCHAR2(25), "PANCOPY" VARCHAR2(1),
"FORM60" VARCHAR2(1));
```

## Unique Key
permits multiple entries of NULL into the column.
clubbed at the top of the column in the order in which they were entered into the table. This is the
essential difference between the Primary Key and the Unique constraints when applied to table column(s).

1. Unique key will not allow duplicate values
2. Unique index is created automatically
3. A table can have more than one Unique key which is not possible in Primary key
4. Unique key can combine upto 16 columns in a Composite Unique key
5. Unique key can not be LONG or LONG RAW data type

### UNIQUE Constraint Defined At The Column Level
```sql 
<ColumnName> <Datatype>(<Size>) UNIQUE
```

### UNIQUE Constraint Defined At The Table Level
```sql
CREATE TABLE TableName
(<ColumnName1> <Datatype>(<Size>), <ColumnName2> < Datatype >(<Size>),
UNIQUE (<ColumnName1>, <ColumnName2>));
```
ex: 
```sql
CREATE TABLE CUST_MSTR (
"CUST_NO" VARCHAR2(10), "FNAME" VARCHAR2(25),
"MNAME" VARCHAR2(25), "LNAME" VARCHAR2(25),
"DOB_INC" DATE, "OCCUP" VARCHAR2(25),
"PHOTOGRAPH" VARCHAR2(25), "SIGNATURE" VARCHAR2(25),
"PANCOPY" VARCHAR2(1), "FORM60" VARCHAR2(1),
UNIQUE(CUST_NO));
```

## Business Rule contraints
Data Integrity Constraints: Ensure that data entered into a system is accurate and consistent. For example, ensuring that a customer's age is within a reasonable range or that a product's price is a positive number.

Validation Rules: Validate input data against specific criteria. This could include format validation (e.g., ensuring an email address is correctly formatted) or logical validation (e.g., ensuring start dates are before end dates).

Mandatory Fields: Specify that certain fields must be populated with data before a record can be saved. For instance, requiring a customer's name and contact information before creating a new account.

Unique Constraints: Ensure that certain data values are unique within a dataset. For example, ensuring that each username in a system is unique to avoid duplicate accounts.

Referential Integrity: Maintain consistency between related data in different tables. For instance, ensuring that a product record cannot be deleted if there are active orders referencing that product.

Business Process Constraints: Define rules that guide how business processes should be executed. This could include approval workflows, escalation rules for unresolved issues, or specific sequences of actions.

Regulatory Compliance: Ensure that data handling and processing adhere to legal and regulatory requirements. This might involve data privacy laws, financial regulations, or industry-specific standards.

# Column Level Constraints
If data constraints are defined as an attribute of a column definition when creating or altering a table structure, they are column level constraints.

# Table Level Constraints
If data constraints are defined after defining all table column attributes when creating or altering a table structure, it is a table level constraint.
    - NOT NULL as column constraint. The NOT NULL column constraint ensures that a table column cannot be left empty.
        When a column is defined as not null, then that column becomes a mandatory column. It implies that a value must be entered into the column if the record is to be accepted for storage in the table.
        Syntax: `<ColumnName> <Datatype>(<Size>) NOT NULL`

# NULL Value Concept
when we don't know what to put in a cell i.e. no such defined type or data we
use null for it-> `INSERT INTO BRANCH_MSTR (BRANCH_NO, NAME) VALUES(BI', null);`

# NOT NULL 
The `NOT NULL` column constraint ensures that a table column cannot be left empty.
```sql
<ColumnName> <Datatype>(<Size>) NOT NULL
```
# CHECK
Business Rule validations can be applied to a table column
CHECK constraints must be specified as a logical expression that evaluates either to TRUE or FALSE.

```sql
<ColumnName> <Datatype>(<Size>) CHECK (<Logical Expression>)
```

ex:
```sql
REATE TABLE CUST_MSTR("CUST_NO" VARCHAR2(10) CHECK(CUST_NO LIKE 'C%'),
"FNAME" VARCHAR2(25) CHECK (FNAME = UPPER(FNAME)),
"WNAME" VARCHAR2(25) CHECK (MNAME = UPPER(MNAME)),
"LNAME" VARCHAR2(25) CHECK (LNAME = UPPER(LNAMB)), "DOB _ INC" DATE,
"QCCUP" VARCHAR2(25), "PHOTOGRAPH" VARCHAR2(25), "SIGNATURE" VARCHAR2(25),
"PANCOPY" VARCHAR2(1), "FORM60" VARCHAR2(1));
```

## At Table Level
```sql
CHECK (<Logical Expression>)
```

ex:
```sql
CREATE TABLE CUST_MSTR("'CUST_NO" VARCHAR2(10), "FNAME" VARCHAR2(25),
"MNAME" VARCHAR2(25), "LNAME" VARCHAR2(25), "DOB_INC" DATE,
"OCCUP" VARCHAR2(25), "PHOTOGRAPH" VARCHAR2(25), "SIGNATURE" VARCHAR2(25),
"PANCOPY" VARCHAR2(1), "FORM60" VARCHAR2(1),
CHECK (CUST_NO LIKE ‘C%’), CHECK (FNAME = UPPER(FNAME)),
CHECK (MNAME = UPPER(MNAME)), CHECK (LNAME = UPPER(LNAME)));
```

Insert operation can be performed to validate: `INSERT INTO CUST_MSTR (CUST_NO, FNAME, MNAME, LNAME, DOB_INC, OCCUP, PHOTOGRAPH,
SIGNATURE, PANCOPY, FORM60)
VALUES('014', 'SHARANAM', 'CHAITANYA‘, 'SHAH’, '03-Jan-1981', Business’, null, null, 'N', 'Y');`


# Restriction/Limitation
- The condition must be a Boolean expression that can be evaluated using the values in the row being inserted or updated.
- The condition cannot contain subqueries or sequences.
- The condition cannot include the SYSDATE, UID, USER or USERENV SQL functions.


