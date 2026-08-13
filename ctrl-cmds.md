# Understanding the SQL\*Plus Environment

> **Dialect note:** Oracle SQL\*Plus *client* settings — they shape the client's output, not the database. They are not SQL and won't run through JDBC/other tools. (Full reference: course PDF, SQL\*Plus chapter.)

### SET SERVEROUTPUT ON — Seeing PL/SQL Output
#### Explain It
PL/SQL prints text via the `DBMS_OUTPUT` package, but the SQL\*Plus client discards that output unless `SET SERVEROUTPUT ON` is active. This is the #1 cause of "my PL/SQL block ran but printed nothing" — the code works, the client just swallows it. `SET SERVEROUTPUT ON SIZE n` also sets the buffer size for long output.

#### Prove It
```sql
SET SERVEROUTPUT ON
BEGIN
  DBMS_OUTPUT.PUT_LINE('hello from serveroutput');
END;
/
```

#### Gotchas / Edge Cases
- Without `SET SERVEROUTPUT ON` the block still executes — only the *printed messages* vanish.
- `SERVEROUTPUT OFF` also disables the buffer; `DBMS_OUTPUT.PUT_LINE` has a 32k-per-line and buffer-size limit (default 20,000 bytes, up to 1,000,000).
- In SQLcl / modern tools the equivalent toggle is `SET SERVEROUTPUT ON` too — but in PL/SQL Developer/TOAD it's a GUI checkbox. Know the client you're in.

---

### SET AUTOTRACE — Execution Plans and Statistics
#### Explain It
`AUTOTRACE` asks SQL\*Plus to append an execution report to DML statements: `EXPLAIN` shows the plan (index vs full scan — the index-tuning tool), `STATISTICS` shows physical reads/consistent gets. `TRACEONLY` suppresses the data rows, which is perfect for big queries where you only want the plan.

#### Prove It
```sql
SET AUTOTRACE ON EXPLAIN
SELECT COUNT(*) FROM ORDERS;      -- prints the execution plan (TABLE ACCESS FULL)
SET AUTOTRACE OFF
```

#### Gotchas / Edge Cases
- AUTOTRACE opens a second session internally on some setups (and needs no plan table in modern Oracle); in old versions it required building a `PLAN_TABLE` first.
- The plan is the optimizer's *estimate* — forcing it ("why didn't my index get used?") is the classic tuning conversation; see indexes in `advanced-sql.md`.
- `SET AUTOTRACE TRACEONLY STATISTICS` is the fastest way to measure a query's fetch cost in an interview-simulation setting.

---

### Substitution Variables (&var / DEFINE)
#### Explain It
SQL\*Plus substitutes `&name` (and `&&name`, which remembers its value for the whole session) with a value *before* sending the statement to the database — the client does this, not SQL. Combined with `DEFINE name = value` you can parameterize scripts interactively, and `VERIFY ON/OFF` toggles the "old/new" echo of the substitution.

#### Prove It
```sql
DEFINE cname = 'B1'
SELECT '&cname' AS SUBS_VALUE FROM DUAL;     -- prints B1
UNDEFINE cname
```
```sql
-- interactively SQL*Plus prompts for &mACCT_NO when the script runs
SELECT * FROM ACCT_MSTR WHERE ACCT_NO = '&mACCT_NO';
```

#### Gotchas / Edge Cases
- `&` inside a string literal also triggers substitution — a literal `&` in data needs the `SET DEFINE OFF` escape.
- `&&` vs `&`: single `&` asks every time, double `&&` reuses the saved value — the interview-ready detail.
- Substitution is lexical: bad values produce SQL errors at *runtime*, not ask-time.

---

### SET LINESIZE — Output Formatting
#### Explain It
LINESIZE is the character width of a printed line; too small and rows wrap mid-column, too large and the terminal wraps. It works with `SET PAGESIZE` (rows per page), `SET NUMWIDTH` (display width of numbers), and `SET NUMFORMAT` (number display format) to make query output readable.

#### Prove It
```sql
SET LINESIZE 40
SELECT BRANCH_NO, NAME FROM BRANCH_MSTR WHERE ROWNUM < 2;   -- narrow columns, wrapped
SET LINESIZE 80
```

#### Gotchas / Edge Cases
- LINESIZE/COLSEP/SPOOL are print-shape levers: for CSV export you combine `SET COLSEP ','` + `SPOOL file` (see `advance-feat.md`).
- These settings affect only display, never the result set — a `FETCH` still returns full values.

---

### Reference Table — SQL\*Plus Environment Variables
The course documents ~38 settable variables; here they are compressed to one line each (the exam-relevant ones are covered above).

| Variable | What it controls |
|---|---|
| `ARRAYSIZE` | Rows fetched per batch (1–5000); bigger batches speed up big fetch queries, more memory |
| `AUTOTRACE` | ON/OFF/TRACEONLY + EXPLAIN/STATISTICS report after DML; TRACEONLY suppresses data rows |
| `BLOCKTERMINATOR` | Character that ends a PL/SQL block (default `.`) |
| `CMDSEP` | Character separating multiple commands on one line; ON = semicolon |
| `CONCAT` | Character terminating a substitution-variable reference (default `.`); OFF to disable |
| `COPYCOMMIT` | After how many batches the COPY command commits |
| `COPYTYPECHECK` | ON/OFF suppression of datatype checks when copying into DB2 |
| `DEFINE` | Character prefixing substitution variables (default `&`); OFF disables scanning |
| `ESCAPE` | Escape character for literals; OFF undefines it |
| `FEEDBACK` | Minimum rows before "n rows selected" is printed; OFF suppresses |
| `LONG` | Max display width of LONG/CLOB/NCLOB values (up to 2 GB) |
| `LONGCHUNKSIZE` | Fetch chunk size for LONG/CLOB values |
| `NUMFORMAT` | Default number display format |
| `NUMWIDTH` | Default number display width |
| `SQLCASE` | Convert commands to UPPER/LOWER/MIXED before execution |
| `SQLBLANKLINES` | ON allows blank lines inside a SQL command |
| `SQLCONTINUE` | Continuation prompt string (default `> `) |
| `SQLNUMBER` | Prompt shows line numbers for continued commands |
| `SQLPREFIX` | Prefix char for inline SQL\*Plus commands inside a command |
| `SQLTERMINATOR` | End-of-statement char (default `;`); OFF = blank line ends it |
| `ECHO` | Whether START scripts echo each command as it runs |
| `EDITFILE` | Default file for the EDIT command |
| `FLAGGER` | FIPS/ANSI conforms-check of statements (SQL92 level) |
| `FLUSH` | OFF buffers output for faster non-interactive runs |
| `INSTANCE` | Which instance subsequent commands talk to |
| `LINESIZE` | Characters per output line |
| `SERVEROUTPUT` | ON shows `DBMS_OUTPUT.PUT_LINE`; SIZE sets the buffer |
| `SUFFIX` | Default file extension for command files (default `SQL`) |
| `SHOWMODE` | ON prints old+new setting on every SET |
| `TAB` | ON formats whitespace with tabs instead of spaces |
| `TERMOUT` | OFF hides command output (used with SPOOL) |
| `TIME` | ON shows clock time before each prompt |
| `TIMING` | ON shows elapsed time per command/P L/SQL block |
| `TRIMOUT` | ON strips trailing blanks from displayed lines |
| `TRIMSPOOL` | ON strips trailing blanks from spooled lines |
| `VERIFY` | ON echoes substitution old/new lines |
| `WRAP` | ON wraps long rows, OFF truncates them |

#### Gotchas / Edge Cases
- None of these persist beyond the session (unless put in the `glogin.sql`/`login.sql` startup script) — the "my SET disappeared" surprise.
- The variable names appear in docs with bracket abbreviations (`ARRAY[SIZE]`, `CMDS[EP]`) — that's the syntax abbreviation, not multiple variables; typing `SET ARRAY 100` works.
- `SET FEEDBACK OFF` is the classic culprit behind "why is my script quiet?" — pair with `SET VERIFY OFF` for clean script output.