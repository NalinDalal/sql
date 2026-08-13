# Grouping Data From SQL Tables

> **Dialect note:** Oracle (ROLLUP syntax, `||` concat).

### GROUP BY
#### Explain It
`GROUP BY` collapses rows into groups that share the same values in the listed columns, so each group can be summarized with an aggregate function (COUNT, SUM, AVG...). Every column in the `SELECT` list must either be one of the grouping columns or appear inside an aggregate — anything else is an error. Rows are grouped *after* `WHERE` filtering, so WHERE cannot see aggregates.

#### Prove It
```sql
SELECT BRANCH_NO, COUNT(ACCT_NO) AS NO_OF_ACCTS
FROM ACCT_MSTR
GROUP BY BRANCH_NO;
```

#### Gotchas / Edge Cases
- "SELECT a column that isn't grouped" → `ORA-00979: not a GROUP BY expression` — the classic compile error interviewers ask about.
- `WHERE` runs before grouping: to filter *groups*, use HAVING (next concept). To filter rows before grouping, use WHERE — do not put both in HAVING.
- NULLs in the grouping column form their own group — a group of "unknown" values appears in the result.

---

### HAVING — Filtering Groups
#### Explain It
HAVING applies a condition to the *groups* created by GROUP BY, keeping only groups whose aggregate result satisfies the test. WHERE filters rows before grouping; HAVING filters groups after aggregation — that ordering is the single most-asked GROUP BY interview question.

#### Prove It
```sql
-- customers holding more than one account/FD
SELECT CUST_NO, COUNT(ACCT_FD_NO) AS NO_OF_ACCTS_HELD
FROM ACCT_FD_CUST_DTLS
WHERE ACCT_FD_NO LIKE 'CA%' OR ACCT_FD_NO LIKE 'SB%'
GROUP BY CUST_NO
HAVING COUNT(ACCT_FD_NO) > 1;

-- customers holding exactly one account/FD
SELECT CUST_NO, COUNT(ACCT_FD_NO) AS NO_HELD
FROM ACCT_FD_CUST_DTLS
GROUP BY CUST_NO
HAVING COUNT(ACCT_FD_NO) = 1;
```

#### Gotchas / Edge Cases
- `WHERE COUNT(...) > 1` is a syntax error — aggregates are not allowed in WHERE; that is HAVING's job.
- HAVING can use columns not in the SELECT list (`HAVING COUNT(*) > 5` works); columns must still be grouped if non-aggregated.
- Both may appear in one query: WHERE strips rows (cheap, early), HAVING strips groups (expensive, late) — pushing filter conditions into WHERE is a classic performance tip.

---

### ROLLUP — Subtotals and Grand Total
#### Explain It
`ROLLUP` extends GROUP BY with subtotal rows: for each grouping level it adds a row with the running aggregate, ending with a grand total row (shown as NULLs in the rolled-up columns). It is the engine's built-in "report with subtotals" — Oracle and Postgres write `GROUP BY ROLLUP(a, b)`; MySQL writes `GROUP BY a, b WITH ROLLUP`.

#### Prove It
```sql
SELECT FD_SER_NO, FD_NO, SUM(AMT), SUM(DUEAMT)
FROM FD_DTLS
GROUP BY ROLLUP (FD_SER_NO, FD_NO);
-- last rows: subtotal per FD_SER_NO, then one grand-total row
```

#### Gotchas / Edge Cases
- Subtotals look like normal rows with NULL in the grouped columns — applications must know to interpret NULLs there as "subtotal", not "missing data".
- `CUBE` (all combinations) and `GROUPING SETS` (chosen combinations) are the siblings of ROLLUP; interviewers sometimes ask "ROLLUP vs CUBE".
- Column order in ROLLUP changes which subtotals you get — `ROLLUP(a, b)` subtotals on a, `ROLLUP(b, a)` on b.

---

### Subqueries (Nested Queries)
#### Explain It
A subquery is a SELECT inside another statement, used to (1) feed values into WHERE/HAVING conditions (`IN`, `=`, `EXISTS`), (2) act as a data source in the FROM clause (an *inline view*), (3) supply rows for INSERT ... SELECT, or (4) build tables/views. Oracle executes non-correlated subqueries once and re-uses the result; correlated subqueries re-execute per outer row.

#### Prove It
```sql
-- 1) IN: an address list only for customers named Ivan Bayross
SELECT CODE_NO AS CUST_NO,
       ADDR1 || ' ' || ADDR2 || ' ' || CITY || ', ' || STATE || ', ' || PINCODE AS ADDRESS
FROM ADDR_DTLS
WHERE CODE_NO IN (SELECT CUST_NO FROM CUST_MSTR
                  WHERE FNAME = 'Ivan' AND LNAME = 'Bayross');

-- 2) FROM clause: inline view (aliases are MANDATORY in Oracle)
SELECT A.ACCT_NO, A.CURBAL, A.BRANCH_NO, B.AVGBAL
FROM ACCT_MSTR A,
     (SELECT BRANCH_NO, AVG(CURBAL) AVGBAL FROM ACCT_MSTR GROUP BY BRANCH_NO) B
WHERE A.BRANCH_NO = B.BRANCH_NO AND A.CURBAL > B.AVGBAL;

-- 3) correlated: same answer as 2, computed per branch
SELECT ACCT_NO, CURBAL, BRANCH_NO FROM ACCT_MSTR A
WHERE CURBAL > (SELECT AVG(CURBAL) FROM ACCT_MSTR WHERE BRANCH_NO = A.BRANCH_NO);

-- 4) multi-column: customers who are also employees (both names match)
SELECT FNAME, LNAME FROM CUST_MSTR
WHERE (FNAME, LNAME) IN (SELECT FNAME, LNAME FROM EMP_MSTR);
```

#### Gotchas / Edge Cases
- An inline view (subquery in FROM) **must have an alias** in Oracle — forgetting it gives `ORA-00933` or "missing alias" errors.
- A subquery feeding `=` or `>` must return exactly one row (an extra row → `ORA-01427: single-row subquery returns more than one row`); for many values use IN or EXISTS.
- Correlated subqueries re-run per outer row — full table scans on both sides can be slow; interview answer: prefer JOIN or a subquery executed once.
- `IN (SELECT ...)` with NULLs in the subquery result matches nothing for the NOT IN case — everything becomes unknown (see `10-interview-questions.md`).