# Grouping Data From SQL Tables

**GROUP BY**
GROUP BY clause is another section of the select statement. 

The `GROUP BY` clause creates a data
set, containing several sets of records grouped together based on a condition.
```sql
SELECT <ColumnName!>, <ColumnName2>, <ColumnNameN>,
AGGREGATE_FUNCTION (<Expression>)
FROM TableName WHERE <Condition>
GROUP BY <ColumnNamel>, <ColumnName2>, <ColumnNameN>;
```

example:
|Tables:|ACCT MSTR|
|-------|---------|
|Columns:|BRANCH_NO, TYPE,_ACCT_NO|
|--------|-------------------------|
|Technique:|Functions: COUNT(), Clauses: GROUP BY, Others: Alias|

```sql
SELECT BRANCH_NO "Branch No.", TYPE "A/C Type", COUNT(ACCT_NO) "No. Of A/Cs"
FROM ACCT MSTR GROUP BY BRANCH _NO, TYPE;
```

**HAVING**
HAVING clause can be used in conjunction with the GROUP BY clause. 
HAVING imposes a condition on the GROUP BY clause, which further filters the groups created by the GROUP BY clause.

Find out the customers having more than one account in the bank.
|Tables:|ACCT_FD_CUS_DTLS|
|Columns:|CUST NO, ACCT_FD NO|
|Technique:|unctions: COUNT(), Operators: LIKE, OR, Clauses: GROUP BY ... HAVING,Others: Alias|

```sql
SELECT CUST_NO, COUNT(ACCT_FD_ NO) "No. Of A/Cs Held" FROM ACCT FD CUST DTLS
WHERE ACCT FD NO LIKE 'CA%' OR ACCT FD NO LIKE 'SB%'
GROUP BY CUST_ NO HAVING COUNT(ACCT FD _NO)>1;
```

determining whether they are unique or not
Snopsis: 
|Tables: ACCT_FD_CUST_DTLS| 
|Columns: CUST_NO.ACCT_FD_NO|
|Technique: Functions: COUNT(), Clauses: GROUP BY ... HAVING, Others: Alias|

```sql
SELECT CUST_NO, COUNT(ACCT_FD NO) "No. Of A/Cs Or FDs Held"
FROM ACCT_FD_CUST_DTLS GROUP BY CUST_ NO HAVING COUNT(ACCT_FD NO) =1;
```

**Group By Using The ROLLUP Operator**
used to calculate aggregates and super aggregates for expressions within a
GROUP BY statement.


|Tables: FD_MSTR |
|Column: FD_SER_NO, FD_NO, AMT, DUEAMT|
|Functions: SUM(), Operators: ROLLUP(), Clauses: GROUP BY|

```sql
SELECT FD_SER_NO, FD_NO, SUM(AMT), SUM(DUEAMT)
FROM FD_DTLS GROUP BY ROLLUP (FD_SER NO, FD_NO);
```

# SUBQUERIES/NESTED Queries
they are queries that appear inside some other query; used for:
- To insert records in a target table
- To create tables and insert records in the table created
- To update records ina target table
- To create views
- To provide values for conditions in WHERE, HAVING, IN and so on used with SELECT, UPDATE, and DELETE statements

|Tables: CUST_MSTR, ADDR_DTLS|
|Columns: CUST_MSTR: CUST_NO, FNAME, LNAME,ADDR_DTLS: CODE NO, ADDR1, ADDR2, CITY, STATE, PINCODE|
|Technique: Sub-Queries, Operators: IN, Clauses: WHERE, Other: Concat(||)|

```sql
SELECT CODE NO "Cust. No.", ADDR1 || '' || ADDR2 ||'' || CITY || ',' || STATE || ', ' || PINCODE "Address"
FROM ADDR_DTLS WHERE CODE_NO IN(SELECT CUST_NO FROM CUST_MSTR
WHERE FNAME = 'Ivan' AND LNAME = 'Bayross');
```

**Using Sub-query In The FROM Clause**
A subquery can be used in the FROM clause of the SELECT statement, 
The concept of using a subquery in the FROM clause of the SELECT statement is called an inline view. 
A subquery in the FROM clause of the SELECT statement defines a data source from that particular Select statement.

|Tables: ACCT_MSTR |
|Columns: ACCT NO, CURBAL, BRANCH_NO|
|Technique: Sub-Queries, Join, Functions: AVG(), Clauses: WHERE, GROUP BY|

```sql
SELECT A.ACCT NO, A.CURBAL, A.BRANCH_NO, B.AVGBAL
FROM ACCT MSTR A, (SELECT BRANCH_NO, AVG(CURBAL) AVGBAL FROM ACCT_MSTR
GROUP BY BRANCH_NO) B
WHERE A.BRANCH NO = B.BRANCH_NO AND A.CURBAL > B.AVGBAL;
```


Using Co-related subqueries:
```sql
SELECT ACCT NO, CURBAL, BRANCH NO FROM ACCT MSTRA
WHERE CURBAL > (SELECT AVG(CURBAL) FROM ACCT MSTR
WHERE BRANCH NO =A.BRANCH NO);
```

Using Multi-Column SubQueries

Synopsis:
Tables:| CUST MSTR, EMP_MSTR
Columns:| CUST MSTR: FNAME, LNAME, EMP_MSTR: FNAME, LNAME
Technique:|CUST MSTR: FNAME, LNAME, EMP_MSTR: FNAME, LNAME

Sub_Queries, Operators: IN, Clauses: WHERE
```sql
SELECT FNAME, LNAME FROM CUST_MSTR
WHERE (FNAME, LNAME) IN(SELECT FNAME, LNAME FROM EMP _ MSTR);
```