# AUDIT — SQL Interview-Prep Repository

Scope: all 15 `.md` files in repo root (readme + 14 topic files).
Method: every section classified as
**(a)** proper explanation exists · **(b)** code/syntax present but weak or no explanation · **(c)** missing (section or explanation absent).
Dialect note and code-validity flags are listed per file. Uncertain/possibly-wrong statements are marked ⚠️ (flag, don't guess).

---

## 1. Overall verdict

| File | Dominant dialect | Dominant grade | Notes |
|---|---|---|---|
| readme.md | — | c | Plain link list; needs module table (STEP 4) |
| 1.intro.md | generic | a | Textbook-style prose; no structure, zero SQL examples |
| 2.installation.md | Postgres **+ Oracle (mixed)** | b | Install steps fine; Tablespace section is Oracle syntax in a Postgres doc |
| ctrl-cmds.md | Oracle (SQL\*Plus) | b | Command-reference list, ~40 variables; terse per-item explanations; ER Model section is an empty placeholder |
| sql-cmd.md | **Oracle, but mixed** (T-SQL `TOP`, psql `\d`) | b | Syntax + examples, thin explanations; several broken snippets |
| constraints.md | Oracle | a/b mix | Good feature lists; multiple garbled/typo'd code blocks; no gotchas |
| computations.md | Oracle | b | Function reference list; one-liners, many non-runnable examples |
| data-grouping.md | Oracle | a | Decent; grouping/subqueries well covered; no gotchas |
| joins.md | ANSI + Oracle (`MINUS`) | a | Best-structured file; **FULL OUTER JOIN missing**; summary table incomplete |
| advanced-sql.md | Oracle | a | Good bullets; ROWID/ROWNUM/SNAPSHOT Oracle-only; B-tree never mentioned |
| advance-feat.md | Oracle | b | 17 sections, each = purpose sentence + pattern; no gotchas |
| oops.md | Oracle | a | Good; object types/nested/VARRAY/REF |
| permissions.md | Oracle | a | Good; no gotchas |
| pl-sql.md | Oracle | a | Good; example uses SQL\*Plus substitution (`&var`) |
| transactions.md | Oracle + **Postgres (`BEGIN;`)** | a | Content strong; one Postgres-style example inside an Oracle doc |
| security.md | Oracle | a | Good; reference style; no gotchas |
| db-obj.md | Oracle | a | Strongest file; procedures-vs-functions table already exists |

**Dialect conclusion:** the repo is based on an Oracle/PL-SQL textbook (Oracle 9i, SQL\*Plus, `DUAL`, `VARCHAR2`, `ROWNUM`, `CONNECT BY`, `MINUS`, `SYNONYM`, packages). Recommendation for STEP 2: **keep Oracle as primary dialect**, annotate it at the top of each file, and convert the few Postgres/T-SQL fragments to Oracle (or label them explicitly as Postgres-only) so examples are consistent and runnable.

---

## 2. Dialect inconsistencies (must fix or flag)

| Location | Snippet | Dialect shown | Problem |
|---|---|---|---|
| sql-cmd.md "Select only limited rows" | `SELECT Top 3 * FROM customers` | T-SQL/SQL Server | Not valid in Oracle/Postgres/MySQL. Oracle: `FETCH FIRST 3 ROWS ONLY`; Postgres: `LIMIT 3` |
| sql-cmd.md "Display table structure" | `\d <TableName>;` | psql (Postgres) | Not SQL; Oracle equivalent: `DESCRIBE` |
| sql-cmd.md "CREATE" | `CREATE SCHEMA "DBA_BANKSYS"; ... SET search_path` | Postgres | Works in Postgres, not Oracle; Oracle uses `CREATE USER` + schema = username |
| sql-cmd.md "Modifying Existing Columns" | `ALTER TABLE ... MODIFY (NAME varchar2(30))` | Oracle | Correct for Oracle, invalid in Postgres/MySQL — fine once dialect is declared |
| transactions.md "Example Workflow" | `BEGIN; ... COMMIT;` | Postgres | Oracle has no `BEGIN;` for transactions (that's PL/SQL block syntax); should be plain statements or annotated |
| installation.md Tablespace | `CREATE [TEMPORARY] TABLESPACE DATAFILE ...` | Oracle-looking, mangled | Matches no real dialect as written ⚠️ (real Oracle syntax differs); Postgres tablespaces use `CREATE TABLESPACE name LOCATION '...'` |
| joins.md Self Join | comma-join theta syntax `FROM EMP_MSTR EMP, EMP_MSTR MNGR` | Oracle/legacy | Valid Oracle+Postgres, NOT valid in MySQL (needs `CROSS JOIN` + `ON`) |
| joins.md Set ops | `MINUS` | Oracle | Valid Oracle/Postgres; MySQL/SQL Server use `EXCEPT` — flag as Oracle-specific |

---

## 3. Per-file section audit

### readme.md
- Link list with 1-line summaries: **(c)** — no module table, no concept lists, no dialect note. Redesign in STEP 4.

### 1.intro.md
| Section | Grade | Notes |
|---|---|---|
| DBMS definition + benefits | a | Clear prose; no structure, no examples (conceptual) |
| RDBMS | a | Too brief to be useful, expand slightly |
| Codd's 12 Rules | a | Long, accurate, well explained — keep, restructure into concepts |
| Normalisation intro & reasons | a | Good |
| 1NF / 2NF / 3NF | a | Good worked example via tables (not SQL — fine for this topic). 3NF table has a typo: `Proj` uses "Employee name" instead of "Project name" ⚠️ |
| SQL features/rules | b | List format; one garbled rule ("SO OaLexical units...") ⚠️ |
| Components (DDL/DML/DCL/DQL) | a/b | Clear list items but zero examples; no Code/DQL example |
| **BCNF** | **c** | Missing entirely — needed for 10-interview-questions.md (normalization question) |
| Gotchas (any) | c | e.g., NULL ≠ empty string (Rule 3 mentions it, no gotcha given) |

### 2.installation.md
| Section | Grade | Notes |
|---|---|---|
| Docker Postgres install | a | Runnable steps; valid Postgres |
| Tablespace | b | Syntax-only, Oracle-style, mangled; explanation ~2 sentences; **dialect conflict with rest of file** |
| ER Model | c | Referenced in readme ("saw sample er model") but only a placeholder line here — content lives nowhere ⚠️ |

### ctrl-cmds.md
| Section | Grade | Notes |
|---|---|---|
| ~40 SQL\*Plus environment variables | b | Each has 1-3 sentence explanation (command-reference style) but no runnable `SET` examples; several OCR typos (`BLO[CKTERMINATOR]`, `ARRAY[SIZE]`, `{15|n}`, `WRA|P]`) ⚠️ |
| ER Model | c | Single placeholder line: "some sample bank accounts model -> name, acct_no..." |
| Gotchas | c | e.g., SET SERVEROUTPUT ON required for DBMS_OUTPUT, AUTOTRACE opens 2nd connection (mentioned inline only) |

### sql-cmd.md
| Section | Grade | Notes |
|---|---|---|
| Data types (CHAR vs VARCHAR) | b | 2 lines; the "CHAR 50% faster" claim is unverifiable folk wisdom ⚠️ |
| CREATE, naming rules | b | Rules ok; example is fine Postgres; dialect conflict |
| INSERT / INSERT SELECT | b | One-liner explanations; `INSERT 0 1` is psql output (Postgres) |
| SELECT variations (WHERE / columns / ordering) | a | WHERE explanation actually decent |
| DISTINCT | a | Good 2-sentence explanation + example |
| TOP / LIMIT rows | b | **T-SQL `Top`** — wrong dialect, must fix |
| CREATE TABLE AS / INSERT SELECT | b | Syntax-only, fine |
| DELETE / UPDATE (+ conditional) | b | Syntax-only; EXISTS-based delete example has mangled identifiers (`ADDR _DTLS`, `CUST MSTR`, smart quote) ⚠️ |
| ALTER / MODIFY / RENAME / TRUNCATE / DROP | b | Syntax + examples; "TRUNCATE not transaction-safe" claim is Oracle-version-dependent ⚠️ (Postgres TRUNCATE IS transactional) |
| Synonyms | a | Good bullet explanation; Oracle-only |
| `\d` | c-b | psql command in an Oracle file; replace with `DESCRIBE` |
| Gotchas | c | No NULL-in-WHERE trap, no WHERE-vs-HAVING pointer, no "quoted identifier case" note (the `"DBA_BANKSYS"."BRANCH_MSTR"` examples will bite in Postgres) |

### constraints.md
| Section | Grade | Notes |
|---|---|---|
| Constraint intro / types | b | 3 lines + taxonomy; ok but thin |
| Primary key | a | Good 9-point feature list; examples have typos (`CUST_ MSTR`, `FD MSTR`, columns named `"FD_SER_NO"` but referenced as `FD_SER NO`) ⚠️ |
| Foreign key | a | Good principles list; garbled rule "Child may have duplicates and nulls but unless it is specified" ⚠️; ON DELETE CASCADE / SET NULL examples fine (Oracle; note Oracle has **no ON UPDATE** clause) |
| Unique key | b | Features list ok, but explanation sentence is garbled ("clubbed at the top of the column in the order in which they were entered") ⚠️ — core fact (multiple NULLs allowed) is correct for Oracle |
| NULL value concept | b | 1 line + example |
| NOT NULL | a | One-liner, fine |
| CHECK | b | Explanation ok; example has `REATE TABLE` typo and a smart-quote ⚠️ |
| Restrictions/limitations (CHECK) | a | Accurate |
| Gotchas | c | No PK-vs-UNIQUE-NULL distinction summary, no composite-key ordering note, no hint that UNIQUE allows multiple NULLs differs by vendor (Oracle: yes; SQL Server: one NULL) |

### computations.md
| Section | Grade | Notes |
|---|---|---|
| Operators + aliases | b | Listing only, no explanation |
| AND / OR / BETWEEN | a | Short but correct; BETWEEN inclusive |
| Aggregate functions (AVG/MIN/COUNT/MAX/SUM) | b | Syntax-only lines; COUNT doesn't mention NULL semantics clearly; no runnable examples |
| Numeric functions | b | Reference-style; most have `FROM DUAL` examples (runnable in Oracle); MOD/TRUNC/FLOOR/CEIL fine |
| String functions | b | Same; several typos (`SELECT LOWER(IVAN BAYROSS')`, `LENGTH('SHARANAM))`, `TRANSLATE(Isct523'...)`, `TRIM(' Hansel ')` example fine) ⚠️ |
| Conversion (TO_NUMBER/TO_CHAR) | b | Examples present; TO_CHAR format strings Oracle-specific (fine once dialect declared) |
| Date functions (ADD_MONTHS, LAST_DAY, MONTHS_BETWEEN...) | b | Examples present, all valid Oracle; `MONTHS_BETWEEN` example shows a typo (`MONTHS BETWEEN` with no underscore) ⚠️ |
| Miscellaneous (UID/USER/SYS_CONTEXT/USERENV/COALESCE) | b | COALESCE explanation good; rest Oracle-session specific |
| Gotchas | c | No NULL-propagation note (SUM ignores NULLs, any arithmetic with NULL → NULL), no implicit-conversion trap, no `FROM DUAL`-vs-`SELECT`-without-FROM dialect note |

### data-grouping.md
| Section | Grade | Notes |
|---|---|---|
| GROUP BY | a | Good explanation + example (typo `ACCT MSTR` / `BRANCH _NO` in code ⚠️) |
| HAVING | a | Good, 2 examples |
| ROLLUP | b | 2-line intro; syntax-only; Oracle-style `GROUP BY ROLLUP(...)` (valid in Oracle/Postgres; MySQL uses `WITH ROLLUP`) |
| Subqueries: IN, inline view (FROM), correlated, multi-column | a | All explained + examples; correlated subquery example valid |
| Gotchas | c | No "WHERE filters rows, HAVING filters groups" ordering rule, no NULL-group behavior, no COUNT(*) vs COUNT(col) vs COUNT(DISTINCT) |

### joins.md *(already best-structured — closest to target format)*
| Section | Grade | Notes |
|---|---|---|
| Inner join | a | Explain + Prove + mermaid |
| Outer/LEFT | a | Explain + Prove |
| **FULL OUTER JOIN** | **c** | Never covered anywhere in repo (not in summary table either) — must be added for 10-interview-questions.md |
| RIGHT OUTER | a | Example is a mirror of LEFT (technically correct); note in gotchas that results equal the LEFT-join version |
| Cross join | a | Correct, with warning about Cartesian products |
| Self join | a | Uses legacy comma syntax (flag dialect) |
| ANSI vs theta syntax | a | Good, but little guidance on when each is preferred |
| Multi-table / WHERE-in-join | a | Fine |
| UNION / INTERSECT / MINUS | a | Good; MINUS Oracle-only (see table above) |
| Gotchas | c | No ON-vs-WHERE filter placement trap for LEFT JOINs, no NULL-key match loss, no many-to-many row explosion note, no "FULL OUTER = LEFT ∪ RIGHT" |

### advanced-sql.md
| Section | Grade | Notes |
|---|---|---|
| Indexes (intro + 8 types + syntax) | a | Good bullets; **B-tree never mentioned** ⚠️; "sequential search stops when value doesn't match" is a simplification |
| Views | a | Good; updateable-view rules solid |
| Clusters | b/a | Oracle-only; fine once dialect declared |
| Sequences | a | Oracle NEXTVAL/CURRVAL syntax correct |
| Snapshots | b | Oracle-era feature (≈ materialized views); syntax `CREATE SNAPSHOT` is valid Oracle but obsolete |
| ROWID / ROWNUM / delete duplicates | a/b | Useful Oracle tricks; ⚠️ claim "ROWNUM is affected by ORDER BY only if index exists" is folklore, flag as uncertain rather than assert |
| Index-damage discussion | b | "too many indexes slow inserts" stated but never demonstrated (no example of a bad index / when not to index) — interview-relevant gap |
| Gotchas | c | No "index helps when: WHERE/ORDER BY/join keys; hurts when: low cardinality, frequent writes, LIKE '%x'" summary |

### advance-feat.md
| Section | Grade | Notes |
|---|---|---|
| All 17 advanced features (CONNECT BY, DECODE, DUMP, ROWNUM tricks, NVL, SOUNDEX, CSV…) | b | Each = 1 purpose line + pattern (mostly runnable Oracle). No explain-depth, no gotchas. Section 2 (DECODE matrix) and 10 reference a `...` ellipsis — not runnable as-is ⚠️ |
| Gotchas | c | e.g., CONNECT BY cycle handling, DECODE ≠ CASE equality-only, SOUNDEX availability |

### oops.md
| Section | Grade | Notes |
|---|---|---|
| Oracle 9i flavours / why objects | a | Good brief prose |
| Abstract datatype / nested types / table usage | a | Correct, runnable Oracle |
| Object views | a | Correct |
| Nested tables & VARRAY | a | Correct; example runnable |
| REFs | a | Correct |
| Gotchas | c | No note that object types are Oracle-specific (Postgres uses CREATE TYPE ... AS (…), MySQL has JSON), no instantiation-vs-constructor traps |

### permissions.md
| Section | Grade | Notes |
|---|---|---|
| GRANT syntax + object privileges + WITH GRANT OPTION | a | Clear and correct (Oracle) |
| Examples / referencing other user's table | a | Runnable |
| REVOKE | a | Correct; "cannot revoke OS-granted privileges" — Oracle-fine |
| Gotchas | c | No grant-option-revocation cascade nuance, no PUBLIC role note, no MySQL `GRANT ... TO 'user'@'host'` dialect contrast |

### pl-sql.md
| Section | Grade | Notes |
|---|---|---|
| Intro, advantages, block structure | a | Good |
| Datatypes, variables, constants | a | Good |
| IF / LOOP / WHILE / FOR | a | All correct, runnable patterns |
| Exceptions | b | 2 lines only |
| Balance-deduction example | a | Runnable in Oracle (uses `&mACCT_NO` substitution) |
| Packages / LOB types | a/b | Brief mentions only |
| Gotchas | c | No `SELECT ... INTO` NO_DATA_FOUND/TOO_MANY_ROWS traps, no VARCHAR2 max length note (4000/32767), no implicit-conversion note |

### transactions.md
| Section | Grade | Notes |
|---|---|---|
| Transactions / COMMIT / ROLLBACK / SAVEPOINT | a | Strong; syntax quick-ref uses Postgres `BEGIN;` — see flags |
| Workflow example | b | `BEGIN;` is Postgres syntax inside an Oracle doc ⚠️ |
| ACID | a | Accurate |
| Isolation levels | a | Accurate for Oracle (READ COMMITTED + SERIALIZABLE only); add Postgres/MySQL matrix in interview file since claim "Read Uncommitted/Repeatable Read not supported" is Oracle-specific |
| Anomalies (dirty/non-repeatable/phantom) | a | Correct; phantom example is Postgres-flavored (`BEGIN;`) but conceptually right |
| Oracle quirks (implicit commit, DDL, locks) | a | Good |
| Gotchas | c | No "DDL commits implicitly" trap is present but buried; no autocommit-tool warning summary |

### security.md
| Section | Grade | Notes |
|---|---|---|
| Concurrency / locking / lock types & levels | a | Correct and detailed (Oracle semantics) |
| Deadlock | a | Correct; example valid |
| Exception handling (named/user-defined/PRAGMA) | a | Correct Oracle syntax |
| Error codes ORA-00054 / ORA-06512 | a | Accurate |
| Gotchas | c | No "SELECT ... FOR UPDATE holds lock until commit" trap summary, no row-vs-table lock escalation warning |

### db-obj.md
| Section | Grade | Notes |
|---|---|---|
| Procedure/function structure, storage, execution steps | a | Good |
| Advantages table | a | Good |
| **Procedures vs Functions table** | a | Already solid — interview file must LINK here, not duplicate |
| CREATE PROCEDURE / FUNCTION syntax | a | Correct Oracle |
| Packages (spec/body/public/private/state) | a | Excellent |
| Overloading + restrictions | a | Correct |
| Gotchas | c | No IN/OUT/IN OUT misuse trap, no "function in WHERE clause per-row cost" note, no DML-restrictions-in-function note (Oracle functions called in SQL can't do DML unless autonomous) |

---

## 4. Content gaps for the interview module (STEP 3 cross-check)

| Required interview topic | Covered where now | Gap |
|---|---|---|
| INNER/LEFT/RIGHT/SELF/CROSS joins | joins.md | **FULL OUTER JOIN missing**; add here + link |
| Normalization 1NF–3NF | 1.intro.md | **BCNF missing**; add as new concept (link to intro for 1NF-3NF) |
| Indexes, B-tree, help/hurt | advanced-sql.md | **B-tree never mentioned**; advanced-sql has types list — add B-tree explanation + when they hurt, link file |
| ACID | transactions.md | Complete — link only |
| Isolation levels | transactions.md | Oracle-only view; add vendor matrix (Postgres: all four; MySQL default REPEATABLE READ), link |
| Primary vs Foreign vs Unique key | constraints.md | Complete — link only (refer to PK/UNIQUE NULL difference) |
| Views vs tables | advanced-sql.md | Covered; interview file adds the "view = stored query" TL;DR + link |
| Stored procedures vs functions | db-obj.md | Complete — link only |

---

## 5. Code-validity summary (STEP 2 must fix)

1. Wrong-dialect statements: `SELECT Top 3` (sql-cmd.md), `BEGIN;` (transactions.md), `\d` (sql-cmd.md), `SET search_path` context (sql-cmd.md) — normalize to Oracle or annotate.
2. Non-runnable examples (typos/OCR damage): constraints.md (`REATE TABLE`, `CUST_ MSTR`, `FD MSTR`, smart quotes), sql-cmd.md (underscore-spaced identifiers: `ADDR _DTLS`, `CUST MSTR`, `ACCT MSTR`, `BRANCH MSTR`), computations.md (broken string literals), data-grouping.md (`BRANCH _NO`, `ACCT_FD CCTV`-style spacing), 1.intro.md (3NF `Proj` table's "Employee name" column), introduced `...` ellipsis in advance-feat.md §2/§10.
3. Uncertain claims to FLAG (not silently keep): "CHAR faster than VARCHAR up to 50%", "ROWNUM affected by ORDER BY only with index", "TRUNCATE not transaction-safe" (Oracle-only truth), "indexes: search stops at first non-match" simplification, garbled unique-key "clubbed at top of column" sentence.

---

## 6. Proposed STEP-2 plan (per file, target format)

Format per concept: `### Concept` / `#### Explain It` / `#### Prove It` (runnable SQL) / `#### Gotchas / Edge Cases`. Dialect header added to each file: `> Dialect: Oracle (SQL/PL-SQL) — [exceptions noted inline]`.

| File | Plan |
|---|---|
| 1.intro.md | Keep prose as Explain It; add small runnable proves where sensible (e.g., DUAL demo for DDL/DML classification); fix 3NF typo; add BCNF concept (new, since interview file needs it — or defer to 10-interview-questions.md; recommend intro) |
| 2.installation.md | Keep install steps; rewrite Tablespace as Explain/Prove for Postgres (*consistent with file's dialect*) with Oracle syntax as a flagged alternative |
| ctrl-cmds.md | Merge ~40 variables into a compact reference table (artifact of a textbook, not interview-relevant) + 3-4 key concepts (SERVEROUTPUT, AUTOTRACE, substitution vars) in target format; fill or delete the ER-Model stub |
| sql-cmd.md | Convert every section to Explain/Prove/Gotchas; fix dialect (TOP→FETCH FIRST, `\d`→DESCRIBE); add NULL-in-WHERE gotcha, WHERE-vs-HAVING pointer |
| constraints.md | Restructure; fix all typos; add gotchas (PK vs UNIQUE NULL, composite-key order, ON DELETE options, no ON UPDATE in Oracle) |
| computations.md | Keep function reference tables (they're complete) but wrap each family in target format + runnable DUAL examples + NULL gotchas |
| data-grouping.md | Group concepts: GROUP BY, HAVING, ROLLUP, subquery types; add WHERE-vs-HAVING gotcha, COUNT(*) vs COUNT(col) |
| joins.md | Already close; add FULL OUTER JOIN concept + gotchas; keep mermaid diagrams |
| advanced-sql.md | Convert index/view/cluster/sequence/snapshot to target format; add B-tree concept (help/hurt) since interview needs it; flag ROWNUM folklore |
| advance-feat.md | Group 17 features into ~6 concepts (hierarchical, DECODE matrices, ROWNUM tricks, date arithmetic, formatting/CSV, NULL-handling, misc) in target format; fix ellipsis examples |
| oops.md | Convert type/nested/VARRAY/REF concepts to target format + gotchas; add dialect note (Oracle-only) |
| permissions.md | Convert GRANT/REVOKE to target format + gotchas |
| pl-sql.md | Convert to target format; add exception gotchas, keep `&var` example with note |
| transactions.md | Convert; fix `BEGIN;`; add gotchas; keep ACID/isolation as concepts (interview links here) |
| security.md | Convert locking/deadlock/exceptions to target format + gotchas |
| db-obj.md | Convert; keep procedures-vs-functions table inside a concept's Explain/Prove; add gotchas |
| 10-interview-questions.md | NEW — 10 questions per STEP 3 spec, linking to the above instead of duplicating |
| readme.md | STEP 4 — module table (topic \| concepts covered \| key file) |

---

*Audit complete — no files modified. Awaiting approval before STEP 2.*