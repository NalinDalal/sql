# Advanced SQL Notes

## SQL Performance Tuning

### Indexes
- **Indexes** are ordered lists of column contents (or groups of columns) used to speed up data retrieval.
- Indexing creates a two-dimensional matrix independent of the table, with sorted data and an address field (ROWID) pointing to the record location.
- When data is inserted, Oracle assigns a unique ROWID for each value, indicating its exact storage location.
- Indexes improve search speed, especially for SELECT statements with WHERE clauses on indexed columns.
- Data in indexes is sorted; sequential search stops when a value does not meet the search criteria.
- Too many indexes can slow down data insertion.

**Types of Indexes:**
- Simple Index: Single column
- Composite Index: Multiple columns
- Unique Index: Enforces uniqueness
- Duplicate Index: Allows duplicates
- Reverse Key Index: Stores values in reverse order
- Bitmap Index: Efficient for columns with few distinct values
- Function-Based Index: Indexes expressions/functions
- Key-Compressed Index: Shares prefix entries for space savings

**Syntax Examples:**
```sql
CREATE INDEX idxName ON TableName(ColumnName);
CREATE UNIQUE INDEX idxName ON TableName(ColumnName);
CREATE INDEX idxName ON TableName(Column1, Column2);
CREATE INDEX idxName ON TableName(ColumnName) REVERSE;
CREATE BITMAP INDEX idxName ON TableName(ColumnName);
CREATE INDEX idxName ON TableName(Function(ColumnName));
CREATE INDEX idxName ON TableName(Column1, Column2) COMPRESS 1;
DROP INDEX idxName;
```

---
## Views
- **Views** are virtual tables created by mapping a SELECT statement to a table or tables.
- Views help reduce redundant data, enforce security, and restrict access to specific columns.
- Views can be read-only or updateable (if based on a single table and include all NOT NULL columns).
- Updateable views allow INSERT, UPDATE, DELETE operations; non-updateable views do not.

**Syntax Examples:**
```sql
CREATE VIEW vwName AS SELECT Column1, Column2 FROM TableName WHERE ...;
DROP VIEW vwName;
```

---
## Clusters
- **Clusters** store data from different tables in the same physical data blocks for performance improvement.
- Cluster key columns define the physical placement of rows.
- Advantages: Reduced disk I/O, improved join performance.
- Disadvantages: Can slow down inserts, columns updated often are not good candidates.

**Syntax Example:**
```sql
CREATE CLUSTER ClusterName (Column DataType);
```

---
## Sequences
- **Sequences** generate numeric values for unique row identification.
- Useful for primary keys and auto-incrementing columns.
- Can specify increment, min/max values, cycle, cache, and order.

**Syntax Example:**
```sql
CREATE SEQUENCE seqName INCREMENT BY 1 START WITH 1 MAXVALUE 999 CYCLE;
SELECT seqName.NEXTVAL FROM DUAL;
ALTER SEQUENCE seqName INCREMENT BY 2;
DROP SEQUENCE seqName;
```

---
## Snapshots
- **Snapshots** are recent copies or subsets of tables, used for distributed databases and fast access.
- Can be simple (single table) or complex (multi-table join).
- Support refresh intervals, primary key, and ROWID options.

**Syntax Example:**
```sql
CREATE SNAPSHOT snapshotName AS SELECT * FROM TableName;
ALTER SNAPSHOT snapshotName ...;
DROP SNAPSHOT snapshotName;
```

---
## Quick Reference Table

| Feature     | Purpose                          | Syntax Example |
|------------|-----------------------------------|---------------|
| Index      | Speed up data retrieval           | CREATE INDEX idxName ON TableName(ColumnName); |
| View       | Virtual table, security, reduce redundancy | CREATE VIEW vwName AS SELECT ... FROM TableName; |
| Cluster    | Store related tables together     | CREATE CLUSTER ClusterName (Column DataType); |
| Sequence   | Generate unique numbers           | CREATE SEQUENCE seqName ...; |
| Snapshot   | Fast access, distributed DBs      | CREATE SNAPSHOT snapshotName AS SELECT ...; |

---
**Tip:** Use indexes wisely—too many can slow down inserts. Views help restrict access and reduce redundancy. Clusters improve join performance. Sequences ensure unique keys. Snapshots enable fast, distributed access.

---
# Deep Dive: Advanced SQL Concepts

## Indexes & ROWID
- **ROWID**: Unique address for each row, not stored in the table, but retrievable via SQL.
- ROWID format (Restricted): BBBBBBBB.RRRR.FFFF (Block, Row, File)
- ROWID format (Extended): OOOOOOFFFFBBBBBBRRRR (Object, File, Block, Row)
- Address field in index points to ROWID, which locates the record in the table.
- Indexes are sorted; search stops at first non-matching value.
- Too many indexes slow inserts.

## Deleting Duplicate Rows
- Use ROWID to delete duplicates, keeping only one row per group:
```sql
DELETE FROM EMP_MSTR WHERE ROWID NOT IN (
	SELECT MIN(ROWID) FROM EMP_MSTR GROUP BY EMP_NO, FNAME, DEPT
);
```
- Use subqueries and GROUP BY to identify unique rows.

## ROWNUM
- ROWNUM assigns a number to each row as it is retrieved.
- Use to limit number of rows:
```sql
SELECT ROWNUM, BRANCH_NO, NAME FROM BRANCH_MSTR WHERE ROWNUM < 4;
```
- ROWNUM is affected by ORDER BY only if index exists on the ordered column.

## Views: Updateable & Master/Detail
- Updateable views: Must be based on a single table, include all NOT NULL columns, and exclude aggregate functions, DISTINCT, GROUP BY, HAVING, subqueries, constants, UNION/INTERSECT/MINUS.
- Error messages for non-updateable views:
	- ORA-01779: cannot modify a column, which maps to a non key-preserved table.
	- ORA-01752: cannot delete from view without exactly one key-preserved table.
	- ORA-01732: data manipulation operation not legal on this view.
- Master/Detail views: Join multiple tables, but INSERT/UPDATE/DELETE only affect the detail table.
```sql
CREATE VIEW vw_Branch AS
SELECT BRANCH_NO, NAME, ADDR_TYPE, ADDR1, ADDR2, CITY, STATE, PINCODE
FROM BRANCH_MSTR, ADDR_DTLS
WHERE ADDR_DTLS.CODE_NO = BRANCH_MSTR.BRANCH_NO;
```

## Index Types: Details
- **Reverse Key Index**: Stores values in reverse byte order, avoids block contention.
- **Bitmap Index**: Efficient for low cardinality columns, space-saving, fast for ad hoc queries.
- **Function-Based Index**: Indexes expressions, not just columns.
- **Key-Compressed Index**: Shares prefix entries, saves space.

## Sequences: Advanced
- Can be reset, concatenated with system date for unique keys.
- Use CURRVAL and NEXTVAL to reference sequence values.
```sql
SELECT seqName.NEXTVAL FROM DUAL;
SELECT seqName.CURRVAL FROM DUAL;
```
- Alter sequence:
```sql
ALTER SEQUENCE seqName INCREMENT BY 2 MAXVALUE 999;
```

## Snapshots: Advanced
- Snapshots can be simple (single table) or complex (multi-table join).
- Support refresh modes: FAST, COMPLETE, FORCE.
- Can be read-only or updateable.
- Syntax for creating and altering:
```sql
CREATE SNAPSHOT snapshotName AS SELECT * FROM TableName;
ALTER SNAPSHOT snapshotName [REFRESH FAST/COMPLETE/FORCE] [START WITH date] [NEXT date];
DROP SNAPSHOT snapshotName;
```

---
## Common Restrictions & Error Messages
- Updateable views: No aggregates, DISTINCT, GROUP BY, HAVING, subqueries, constants, UNION/INTERSECT/MINUS, or views based on other non-updateable views.
- Error messages:
	- ORA-01779: cannot modify a column, which maps to a non key-preserved table.
	- ORA-01752: cannot delete from view without exactly one key-preserved table.
	- ORA-01732: data manipulation operation not legal on this view.

---
## Syntax Quick Reference (Expanded)
| Feature     | Purpose                          | Syntax Example |
|------------|-----------------------------------|---------------|
| Index      | Speed up data retrieval           | CREATE INDEX idxName ON TableName(ColumnName); |
| Unique Index | Enforce uniqueness              | CREATE UNIQUE INDEX idxName ON TableName(ColumnName); |
| Bitmap Index | Low cardinality, space-saving   | CREATE BITMAP INDEX idxName ON TableName(ColumnName); |
| Reverse Key Index | Avoid block contention      | CREATE INDEX idxName ON TableName(ColumnName) REVERSE; |
| Function-Based Index | Index expressions        | CREATE INDEX idxName ON TableName(Function(ColumnName)); |
| View       | Virtual table, security, reduce redundancy | CREATE VIEW vwName AS SELECT ... FROM TableName; |
| Updateable View | Data manipulation            | CREATE VIEW vwName AS SELECT ... FROM TableName; |
| Master/Detail View | Join tables                | CREATE VIEW vw_Branch AS SELECT ... FROM ... WHERE ...; |
| Cluster    | Store related tables together     | CREATE CLUSTER ClusterName (Column DataType); |
| Sequence   | Generate unique numbers           | CREATE SEQUENCE seqName ...; SELECT seqName.NEXTVAL FROM DUAL; |
| Snapshot   | Fast access, distributed DBs      | CREATE SNAPSHOT snapshotName AS SELECT ...; |
| Delete Duplicates | Remove duplicate rows       | DELETE FROM Table WHERE ROWID NOT IN (SELECT MIN(ROWID) ...); |
| ROWNUM     | Limit rows in query               | SELECT ROWNUM, ... FROM Table WHERE ROWNUM < N; |

---
## Tips & Best Practices
- Use indexes for frequently queried columns, but avoid over-indexing.
- Use views to restrict access and reduce redundancy.
- Use ROWID and ROWNUM for advanced data manipulation.
- Use sequences for unique keys, especially in distributed systems.
- Use snapshots for fast access and distributed environments.