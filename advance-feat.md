# Advanced Features in SQL*Plus

## 1. Tree-Structured Queries (Hierarchical Queries)
- **Purpose:** Extract hierarchical relationships (e.g., employee-manager trees).
- **Pattern:**
  ```sql
  SELECT LPAD(' ', LEVEL * 4) || FNAME || ' ' || LNAME "Employee Hierarchy"
    FROM EMP_MSTR
   CONNECT BY PRIOR EMP_NO = MNGR_NO
   START WITH MNGR_NO IS NULL;
  ```
- **Notes:** Uses `LEVEL`, `CONNECT BY PRIOR`, and `START WITH` for hierarchy. Oracle-specific.

---

## 2. Matrix Reports with DECODE
- **Purpose:** Display data in a matrix format (e.g., branch vs. employee count).
- **Pattern:**
  ```sql
  SELECT B.NAME "BRANCH",
         DECODE(E.BRANCH_NO, 'B1', COUNT(E.EMP_NO)) "B1",
         DECODE(E.BRANCH_NO, 'B2', COUNT(E.EMP_NO)) "B2",
         ...
    FROM EMP_MSTR E, BRANCH_MSTR B
   WHERE B.BRANCH_NO = E.BRANCH_NO
   GROUP BY B.NAME, E.BRANCH_NO;
  ```
- **Notes:** `DECODE` substitutes values for matrix output.

---

## 3. Examine Column Contents (DUMP)
- **Purpose:** View internal column details (type, length, ASCII values).
- **Pattern:**
  ```sql
  SELECT DUMP(ACCT_NO) FROM ACCT_MSTR;
  ```

---

## 4. Drop Columns (Workarounds)
- **Purpose:** Remove columns (pre-Oracle 8i).
- **Patterns:**
  - Set column to NULL, rename table, create view without column.
  - Create new table without column, drop old table, rename new table.
  - Oracle 8i+: `ALTER TABLE ... DROP COLUMN ...;`

---

## 5. Rename Columns
- **Purpose:** Change column names.
- **Patterns:**
  - Add new column, copy data, drop old column.
  - Create new table with desired column names, copy data, drop old table, rename.

---

## 6. Select Every Nth Row
- **Purpose:** Retrieve every Nth row (e.g., even rows).
- **Patterns:**
  ```sql
  SELECT ROWNUM, EMP_NO, FNAME FROM EMP_MSTR WHERE MOD(ROWNUM, 2) = 0;
  ```

---

## 7. Generate Primary Keys
- **Purpose:** Populate primary key column.
- **Patterns:**
  - Use `ROWNUM` for simple keys.
  - Use sequence generator:
    ```sql
    CREATE SEQUENCE SEQ_CUSTNO START WITH 1 INCREMENT BY 1;
    UPDATE CUSTOMERS SET CUST_NO = SEQ_CUSTNO.NEXTVAL;
    ```

---

## 8. Date Arithmetic
- **Purpose:** Add days/hours/minutes/seconds to dates.
- **Patterns:**
  ```sql
  SELECT SYSDATE + 1/24 FROM DUAL; -- Add 1 hour
  SELECT SYSDATE + 1 FROM DUAL;    -- Add 1 day
  ```

---

## 9. Count Distinct Values
- **Purpose:** Count unique values in a column.
- **Pattern:**
  ```sql
  SELECT ACCT_NO, COUNT(*) FROM TRANS_MSTR GROUP BY ACCT_NO;
  ```

---

## 10. Matrix Reports with DECODE and GROUP BY
- **Purpose:** Summarize multiple categories in one report.
- **Pattern:**
  ```sql
  SELECT CUST_NO,
         SUM(DECODE(SUBSTR(ACCT_FD_NO, 1, 2), 'CA', 1, 0)) "CURRENT ACCOUNTS",
         SUM(DECODE(SUBSTR(ACCT_FD_NO, 1, 2), 'SB', 1, 0)) "SAVINGS ACCOUNTS",
         SUM(DECODE(SUBSTR(ACCT_FD_NO, 1, 2), 'FS', 1, 0)) "FIXED DEPOSITS",
         COUNT(ACCT_FD_NO) "TOTAL"
    FROM ACCT_FD_CUST_DTLS
   GROUP BY CUST_NO;
  ```

---

## 11. Retrieve Rows X to Y
- **Purpose:** Get records between two row numbers.
- **Pattern:**
  ```sql
  SELECT * FROM (SELECT ROWNUM RN, FNAME FROM EMP_MSTR WHERE ROWNUM < 8) WHERE RN BETWEEN 4 AND 7;
  ```

---

## 12. Change Oracle Password
- **Purpose:** Update user password.
- **Pattern:**
  ```sql
  ALTER USER hansel IDENTIFIED BY hansel123;
  ```

---

## 13. Add Line Feeds to SELECT Output
- **Purpose:** Format output with newlines.
- **Pattern:**
  ```sql
  SELECT 'CUSTOMER NAME: ' || FNAME || CHR(10) || 'BIRTHDATE: ' || DOB_INC || CHR(10) FROM CUST_MSTR;
  ```

---

## 14. Numeric to Alphabets (Julian Date Conversion)
- **Purpose:** Convert numbers to words.
- **Pattern:**
  ```sql
  SELECT TO_CHAR(TO_DATE(34654,'J'),'JSP') FROM DUAL;
  ```

---

## 15. CSV Output
- **Purpose:** Export query results as CSV.
- **Pattern:**
  ```sql
  SET COLSEP ','
  SPOOL MY_EMP_REPORT.TXT
  SELECT ... FROM ...;
  SPOOL OFF
  ```

---

## 16. Replace NULLs with Meaningful Values
- **Purpose:** Substitute NULLs in output.
- **Pattern:**
  ```sql
  SELECT NVL(FNAME, 'A'), NVL(MNAME, 'Corporate'), NVL(LNAME, 'Customer') FROM CUST_MSTR;
  ```

---

## 17. Soundex-Based Queries
- **Purpose:** Phonetic matching for similar-sounding names.
- **Pattern:**
  ```sql
  SELECT * FROM MyFriends WHERE SOUNDEX(NAME) = SOUNDEX('Nita');
  ```