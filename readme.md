# SQL Interview Prep — Module Map

> **Dialect note:** All files in this repo use **Oracle/PL-SQL** as the primary dialect, except `2.installation.md` which uses Postgres for local setup steps (noted at the top of that file).

---

## Module Table

| Topic                                        | Concepts covered                                                               | Key file              |
| -------------------------------------------- | ------------------------------------------------------------------------------ | --------------------- |
| Introduction and Normalisation                | 1NF, 2NF, 3NF, BCNF; SQL basics; data vs DB vs RDBMS                          | `1.intro.md`          |
| Local Installation                           | Postgres install (Docker), user/db creation, tablespaces                       | `2.installation.md`   |
| SQL Commands                                 | DDL, DML, DCL, DQL, TCL; `FETCH FIRST`, `DESCRIBE`, schema model               | `sql-cmd.md`          |
| Constraints                                  | PK, UNIQUE, FK, NOT NULL, CHECK; column-vs-table level; NULL behaviour         | `constraints.md`      |
| Computations                                 | Arithmetic, aggregates, numeric/string/date/conversion functions, COALESCE      | `computations.md`     |
| Data Grouping                                | GROUP BY, HAVING, ROLLUP, subqueries (IN, inline view, correlated)             | `data-grouping.md`    |
| Joins                                        | INNER, LEFT, RIGHT, FULL OUTER, CROSS; ANSI vs theta; set ops (UNION/INTERSECT/MINUS) | `joins.md`       |
| Advanced SQL                                 | B-tree indexes (when to use and avoid), views, clusters, sequences, snapshots/MVs, ROWID, ROWNUM | `advanced-sql.md` |
| Advanced Features                            | DECODE, NVL, SOUNDEX, hierarchical queries, date arithmetic, ALTER USER, JSP   | `advance-feat.md`     |
| OOPs in SQL                                  | Object types, VARRAYs, nested tables, REFs, object views                       | `oops.md`             |
| PL/SQL Basics                                | Block structure, variables, control structures, cursors, exception handling     | `pl-sql.md`           |
| Transactions                                 | COMMIT/ROLLBACK/SAVEPOINT, ACID, vendor isolation-level matrix, anomalies       | `transactions.md`     |
| Security Management                          | GRANT/REVOKE, roles, PUBLIC, WITH GRANT OPTION, privilege chains                | `permissions.md`      |
| Locking, Deadlock, Exceptions                | Row/table locks, SELECT FOR UPDATE, deadlock detection, named/user-defined exceptions | `security.md` |
| Database Objects                             | Procedures, functions, packages, overloading, package state                     | `db-obj.md`           |
| SQL*Plus Environment                         | `SERVEROUTPUT`, `AUTOTRACE`, substitution variables, `LINESIZE`; variable reference table | `ctrl-cmds.md` |
| 10 Interview Questions                       | Normalisation, joins, indexes, isolation levels, constraints, procedures, TRUNCATE, ROWNUM, GROUP BY, DECODE | `10-interview-questions.md` |

---

## Quick Links

- [Video course (30 hrs)](https://www.youtube.com/watch?v=SSKVgrwhzus&list=WL&index=1&t=1533s) — do side-by-side
- [Official Oracle SQL Reference](https://www.postgresql.org/docs/current/sql.html)
- [Video resources](https://github.com/DataWithBaraa/sql-ultimate-course)
- [Dataset](https://github.com/DataWithBaraa/sql-ultimate-course/tree/main/datasets/sql-server)
