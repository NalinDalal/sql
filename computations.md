# Computations in SQL

> **Dialect note:** Oracle — the demo table `DUAL` is Oracle's one-row dummy table; PostgreSQL/MySQL use `SELECT <expr>;` without FROM.

### Arithmetic Operators and Precedence
#### Explain It
SQL supports `+`, `-`, `*`, `/`, exponentiation (`**` in Oracle, `^` in MySQL/Postgres), and parentheses for grouping. Parentheses override the normal precedence, and mixing operands with NULLs yields NULL — there is no "0 for missing" arithmetic.

#### Prove It
```sql
SELECT 2 + 3 * 4 FROM DUAL;      -- 14 (multiplication first)
SELECT (2 + 3) * 4 FROM DUAL;    -- 20 (parentheses win)
SELECT AMOUNT, AMOUNT * 1.18 AS WITH_GST FROM ORDERS WHERE ROWNUM < 2;
```
```sql
SELECT NULL + 1000 FROM DUAL;    -- NULL, not 1000
```

#### Gotchas / Edge Cases
- Any arithmetic with NULL propagates NULL: `CURBAL + 1000` on a NULL balance stays NULL — defend against it with `NVL`/`COALESCE` (see the last concept in this file).
- Integer division: Oracle `7 / 2` = 3.5; MySQL gives 3.5 too, but older engines (and `DIV` in MySQL) can truncate — say "it depends on the engine" if asked.
- Oracle's `**` exponentiation is not portable; `POWER()` is the portable form.

---

### Renaming Columns (Aliases)
#### Explain It
Aliases give a column or expression a display name in the result set. They make computed expressions readable (`AMOUNT * 1.18 AS WITH_GST`). Quoted aliases preserve spaces and exact case; unquoted ones are uppercase in Oracle.

#### Prove It
```sql
SELECT AMOUNT AS PRICE, AMOUNT * 1.18 AS "Price with GST"
FROM ORDERS WHERE ROWNUM < 2;
```

#### Gotchas / Edge Cases
- An alias with spaces/case must be double-quoted (`"Price with GST"`) or the space breaks the statement.
- `AS` is optional (`SELECT AMOUNT PRICE ...`) — many style guides require it for clarity.
- An alias cannot be used in `WHERE` (WHERE runs before the SELECT list), but it CAN be used in `ORDER BY`.

---

### Logical Operators: AND / OR
#### Explain It
`AND` requires every condition to be true for a row to match; `OR` returns a row when any one condition matches. Combining them in one WHERE needs parentheses — `AND` binds tighter than `OR`, so without parentheses you get a different (usually wrong) result.

#### Prove It
```sql
SELECT * FROM ORDERS WHERE AMOUNT < 13000 AND AMOUNT > 0;   -- both must hold
SELECT * FROM ORDERS WHERE AMOUNT = 13000 OR AMOUNT > 0;    -- either holds
SELECT * FROM ORDERS WHERE (AMOUNT > 1000 OR ITEM = 'Mouse') AND CUSTOMER_ID = 20;
```

#### Gotchas / Edge Cases
- `A AND B OR C` parses as `(A AND B) OR C` — the classic ASK-IS-IT-checked priority trap; always parenthesize.
- A NULL condition makes OR/AND behave surprisingly: `WHERE AMOUNT > 1000 OR AMOUNT <= 1000` still *excludes* rows where AMOUNT is NULL (NULL is neither > nor <=).

---

### Range Searching with BETWEEN
#### Explain It
`BETWEEN low AND high` is shorthand for `>= low AND <= high` — the range is **inclusive at both ends**, which catches people who assume exclusive bounds. It works on numbers, dates, and formatted strings.

#### Prove It
```sql
SELECT * FROM TRANS_MSTR
WHERE TO_CHAR(DT, 'MM') BETWEEN '01' AND '03';   -- transactions of Jan-Mar
```

#### Gotchas / Edge Cases
- Inclusive ends: `BETWEEN 0 AND 100` includes 100.
- On strings, comparison is character-based: `BETWEEN '01' AND '03'` matches '01','02','03' but also '0X'-style values — prefer dates or numbers for ranges.
- `NOT BETWEEN` inverts the range but NULLs are still excluded from both sides.

---

### Aggregate Functions
#### Explain It
Aggregates collapse a set of values into one number. `SUM`, `AVG`, `MIN`, `MAX` ignore NULLs; `COUNT(*)` counts rows, `COUNT(expr)` counts **non-NULL** values of expr, and `COUNT(DISTINCT expr)` counts distinct non-NULL values. They are the functions that pair with `GROUP BY` (see `data-grouping.md`).

#### Prove It
```sql
SELECT AVG(AMOUNT), MIN(AMOUNT), MAX(AMOUNT), SUM(AMOUNT)
FROM ORDERS;

SELECT COUNT(*) FROM ORDERS;             -- all rows
SELECT COUNT(AMOUNT) FROM ORDERS;        -- rows where AMOUNT is not NULL
SELECT COUNT(DISTINCT ITEM) FROM ORDERS; -- distinct item names
```

#### Gotchas / Edge Cases
- `COUNT(*)` vs `COUNT(col)` is a top interview pair: columns with NULLs make the difference.
- `SUM` over zero rows or all-NULL rows returns NULL, not 0 — wrap with `NVL(SUM(x), 0)` for report totals.
- `AVG` ignores NULLs rather than treating them as 0 — the numbers look "wrong" if you expected (sum/count of all rows).
- Aggregates cannot appear in a `WHERE` clause (that's `HAVING`'s job — `data-grouping.md`).

---

### Numeric Functions
#### Explain It
Oracle's numeric functions: `ABS` (absolute), `POWER(m,n)` (m^n), `ROUND(n[,m])` (round to m decimals, default 0), `SQRT`, `EXP` (e^n), `MOD(m,n)` (remainder; returns m when n is 0), `TRUNC(n[,m])` (cut, not round — works with negative m), `FLOOR` (round down), `CEIL` (round up), `GREATEST`/`LEAST` (largest/smallest across a list), and `EXTRACT` (pull a date part).

#### Prove It
```sql
SELECT ABS(-15) FROM DUAL;                 -- 15
SELECT POWER(2,4) FROM DUAL;               -- 16
SELECT ROUND(15.19,1) FROM DUAL;           -- 15.2
SELECT SQRT(25) FROM DUAL;                 -- 5
SELECT EXP(5) FROM DUAL;                   -- 148.413...
SELECT MOD(15,7) FROM DUAL;                -- 1
SELECT MOD(15,0) FROM DUAL;                -- 15  (Oracle quirk)
SELECT TRUNC(125.815,1) FROM DUAL;         -- 125.8
SELECT TRUNC(125.815,-2) FROM DUAL;        -- 100 (truncates the tens digit)
SELECT FLOOR(24.8), CEIL(24.8) FROM DUAL;  -- 24, 25
SELECT GREATEST(4,5,17), LEAST(4,5,17) FROM DUAL;   -- 17, 4
SELECT EXTRACT(YEAR FROM DATE '2004-07-02') FROM DUAL;   -- 2004
```

#### Gotchas / Edge Cases
- `TRUNC` cuts toward zero, `FLOOR` rounds down: `TRUNC(-1.5)` = -1 but `FLOOR(-1.5)` = -2.
- `ROUND` vs `TRUNC` on negative scale (`-2`) behaves differently for large numbers — know which one you're doing.
- `GREATEST`/`LEAST` with strings compares alphabetically but with implicit conversion surprises if types mix (`GREATEST('4','5','17')` is string compare → '5').

---

### String Functions
#### Explain It
Oracle's string toolbox: case control `LOWER`/`UPPER`/`INITCAP`, extraction and search `SUBSTR`/`INSTR`/`LENGTH`, padding `LPAD`/`RPAD`, trimming `LTRIM`/`RTRIM`/`TRIM`, character codes `ASCII`, single-char replacement `TRANSLATE`, byte size `VSIZE`, and Unicode composers `COMPOSE`/`DECOMPOSE`. Positions start at 1, not 0.

#### Prove It
```sql
SELECT LOWER('IVAN BAYROSS') FROM DUAL;         -- ivan bayross
SELECT INITCAP('ivan bayross') FROM DUAL;       --  Ivan Bayross
SELECT UPPER('Ms. Carol') FROM DUAL;            --  MS. CAROL
SELECT SUBSTR('SECURE',3,4) FROM DUAL;          --  CURE
SELECT INSTR('SCT on the net','t',1,2) FROM DUAL; -- 13 (2nd 't')
SELECT LENGTH('SHARANAM') FROM DUAL;            --  8
SELECT LTRIM('NISHA','N'), RTRIM('SUNILA','A') FROM DUAL;  -- ISHA, SUNIL
SELECT TRIM('  Hansel  ') FROM DUAL;            --  Hansel (no spaces)
SELECT LPAD('Page 1',10,'*') FROM DUAL;         --  ****Page 1
SELECT TRANSLATE('isct523','123','7a9') FROM DUAL;  -- isct7a9
SELECT ASCII('a'), VSIZE('SCT on the net') FROM DUAL;  -- 97, 15
SELECT COMPOSE('a' || UNISTR('\0301')) FROM DUAL;   -- á (a + combining accent)
```

#### Gotchas / Edge Cases
- `SUBSTR` counts from 1, and a **negative start** counts from the end (`SUBSTR('SECURE',-3)` = 'URE').
- `TRANSLATE` replaces character-by-character; `REPLACE` swaps whole strings — two different functions, a common interview mix-up.
- `INSTR` returns 0 (not NULL) when the substring is absent; `LENGTH` on a NULL is NULL, and on a `CHAR` column counts the padded spaces — trim first if the count looks long.

---

### Conversion Functions (TO_NUMBER / TO_CHAR / TO_DATE)
#### Explain It
Conversions move values between character, numeric, and date types. `TO_CHAR` renders numbers and dates as formatted strings (`'$099,999'`, `'Month DD, YYYY'`), `TO_NUMBER` parses text into numbers, and `TO_DATE` parses a string into a date using a format model you supply. Oracle converts implicitly in many places, but explicit conversion is predictable and recommended.

#### Prove It
```sql
SELECT TO_CHAR(17145, '$099,999') FROM DUAL;              -- $017,145
SELECT TO_CHAR(SYSDATE, 'Month DD, YYYY') FROM DUAL;      -- e.g. August 13, 2026
SELECT TO_NUMBER('100') + 1 FROM DUAL;                    -- 101
SELECT TO_DATE('25-JUN-1952', 'DD-MON-YYYY') FROM DUAL;   -- 1952-06-25
```

#### Gotchas / Edge Cases
- Format-model chars (`MM`, `DD`, `YYYY`, `$099,999`) are Oracle-specific but conceptually similar everywhere — PostgreSQL uses `to_char(x,'FM999.99')`, MySQL `DATE_FORMAT`.
- Implicit conversion between strings and dates follows the session's `NLS_DATE_FORMAT` — a string that parses on one machine can fail on another; explicit `TO_DATE` removes the surprise.
- `TO_NUMBER('100')` fails (ORA-01722) if the string has stray characters — validate before converting.

---

### Date Functions and Arithmetic
#### Explain It
Oracle stores dates with a time component and lets you do arithmetic in *days*: `SYSDATE + 1` is tomorrow, `SYSDATE + 1/24` is one hour later. Dedicated functions: `ADD_MONTHS`, `LAST_DAY` (month end), `MONTHS_BETWEEN` (returns fractional months), `NEXT_DAY` (next given weekday), `ROUND`/`TRUNC` on dates, and `NEW_TIME` (time-zone convert a stored moment).

#### Prove It
```sql
SELECT SYSDATE + 1 FROM DUAL;                 -- tomorrow
SELECT SYSDATE + 1/24 FROM DUAL;              -- one hour from now
SELECT ADD_MONTHS(SYSDATE, 4) FROM DUAL;      -- four months ahead
SELECT LAST_DAY(SYSDATE) FROM DUAL;           -- last day of current month
SELECT MONTHS_BETWEEN(TO_DATE('02-FEB-92','DD-MON-YY'),
                      TO_DATE('02-JAN-92','DD-MON-YY')) FROM DUAL;  -- 1
SELECT NEXT_DAY(SYSDATE, 'Saturday') FROM DUAL;       -- next Saturday
SELECT ROUND(TO_DATE('01-JUL-04','DD-MON-YY'),'YYYY') FROM DUAL;     -- 01-JAN-05
SELECT NEW_TIME(TO_DATE('2004/07/01 01:45','yyyy/mm/dd HH24:MI'),
                'AST','MST') FROM DUAL;             -- 23:45 the day before
```

#### Gotchas / Edge Cases
- Date arithmetic is per-day: `added hours = fraction of a day`; forgetting `1/24` is the classic bug.
- `MONTHS_BETWEEN(d1,d2)` is negative when d1 < d2 and returns fractions — don't `ROUND` blindly.
- `NEXT_DAY`'s weekday name depends on the session language ('Saturday' vs 'SAMEDI') — another NLS dependency to flag in interviews.
- `NEW_TIME` is legacy and doesn't know DST changes; modern Oracle prefers `AT TIME ZONE` / `FROM_TZ` — mention this if asked about timezone handling.

---

### Session Functions (USER / UID / SYS_CONTEXT / USERENV)
#### Explain It
These report on *who you are* in the current session: `USER` returns the connected username, `UID` its numeric id, `USERENV('LANGUAGE')` the session language settings, and `SYS_CONTEXT('USERENV', '<param>')` any of dozens of session attributes (NLS settings, database name, etc.). They are neat for audit columns (`created_by` defaults).

#### Prove It
```sql
SELECT USER FROM DUAL;                            -- BANK
SELECT UID FROM DUAL;                             -- e.g. 500
SELECT USERENV('LANGUAGE') FROM DUAL;             -- e.g. AMERICAN_AMERICA.AL32UTF8
SELECT SYS_CONTEXT('USERENV','NLS_DATE_FORMAT') FROM DUAL;  -- session date format
```

#### Gotchas / Edge Cases
- `SYS_CONTEXT` parameters are case-sensitive and there are many — a wrong name returns NULL, not an error.
- `UNISTR`/`USERENV` are Oracle-only; PostgreSQL uses `current_user`/`pg_backend_pid()`, MySQL `USER()`.
- `UID` is deprecated in favor of `USERENV('SESSIONID')` in modern Oracle — it still runs but don't build new code on it.

---

### COALESCE — First Non-NULL Value
#### Explain It
`COALESCE(a, b, c, ...)` walks the argument list and returns the first non-NULL one; if every argument is NULL, it returns NULL. It is the standard portable NULL-substitution tool — the IF-THEN-ELSE chain you'd otherwise hand-write. `NVL` is Oracle's two-argument special case of the same idea.

#### Prove It
```sql
SELECT CUST_NO, COALESCE(FNAME, 'NO NAME GIVEN') AS DISPLAY_NAME
FROM CUST_MSTR;
-- equivalent IF-THEN-ELSE:
--    IF FNAME IS NOT NULL THEN FNAME ELSE 'NO NAME GIVEN'
SELECT NVL(MNAME, 'Corporate') FROM CUST_MSTR WHERE ROWNUM < 2;
```

#### Gotchas / Edge Cases
- `COALESCE` evaluates arguments until a non-NULL is found (short-circuit); `NVL` evaluates its second argument always — relevant if arguments are expensive function calls.
- Both only rescue NULL, not the empty string: in Oracle `''` IS NULL so COALESCE catches it — but in PostgreSQL `''` is a real string and passes through.
- A `COUNT`-style gotcha: `NVL(SUM(x), 0)` is the idiom for "print 0 instead of blank" in report totals.