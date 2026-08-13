# Security Management Using SQL

> **Dialect note:** Oracle syntax throughout. Postgres uses `GRANT ... ON TABLE` for DML and `\dp` for privilege inspection; MySQL uses `SHOW GRANTS FOR user`. Differences are noted in gotchas.

---

### Object Privileges (GRANT / REVOKE)

#### Explain It

Objects created by a user are **owned and controlled** by that user. To let another user access those objects, the owner must explicitly grant privileges. Privileges come in two categories:

- **System Privileges** — broad rights like `CREATE SESSION`, `CREATE TABLE`, `CREATE VIEW`. Granted without `ON <object>`.
- **Object Privileges** — rights on a specific object: `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `ALTER`, `INDEX`, `REFERENCES`, `EXECUTE` (for procedures/functions/packages).

`WITH GRANT OPTION` lets the grantee pass the privilege further to other users — creating a **privilege chain**.

#### Prove It

```sql
-- Grant SELECT on a table to a user
GRANT SELECT ON emp_mstr TO hansel;

-- Grant multiple privileges
GRANT SELECT, UPDATE, INSERT ON emp_mstr TO ivan;

-- Grant with grant option (privilege chain begins)
GRANT SELECT ON emp_mstr TO chhaya WITH GRANT OPTION;

-- Grant EXECUTE on a procedure
GRANT EXECUTE ON emp_mgmt TO PUBLIC;

-- Reference another user's table (prefix with owner)
SELECT * FROM hr.emp_mstr WHERE dept_id = 10;

-- Check current user's granted privileges
SELECT * FROM user_tab_privs;

-- Check system privileges
SELECT * FROM user_sys_privs;
```

#### Gotchas / Edge Cases

- `REVOKE` only removes the **direct grant** — it does **not cascade** to users who received the privilege via `WITH GRANT OPTION`. If A grants to B (with option), and B grants to C, revoking from A leaves C's privilege intact. You must explicitly `REVOKE` from each grantee in the chain.
- Granting to `PUBLIC` gives the privilege to **every current and future user** — it is almost never appropriate in production and is hard to audit out.
- `WITH GRANT OPTION` on `UPDATE` applies only to **existing rows** — the grantee cannot grant update access to future rows (column-level `UPDATE(col)` is needed for that).
- `REVOKE` cannot remove privileges granted via the **operating system** (only Oracle-level grants can be revoked here)

---

### Revoking Privileges

#### Explain It

`REVOKE` removes object privileges that were previously granted. It applies only to Oracle-level grants — privileges granted via OS-level mechanisms (e.g., OS authentication) cannot be revoked with this statement.

`REVOKE ALL` removes all object privileges on the named object from the named user in one statement.

#### Prove It

```sql
-- Revoke a single privilege
REVOKE DELETE ON emp_mstr FROM anil;

-- Revoke all object privileges on a table
REVOKE ALL ON emp_mstr FROM anil;

-- Revoke SELECT on another user's table
REVOKE SELECT ON hr.emp_mstr FROM rocky;

-- Revoke EXECUTE on a procedure
REVOKE EXECUTE ON emp_mgmt FROM PUBLIC;

-- Verify revocation
SELECT * FROM user_tab_privs WHERE grantee = 'ANIL';
```

#### Gotchas / Edge Cases

- Revoking `SELECT` **does not** invalidate currently-open cursors — the holder can continue reading until the cursor is closed
- Revoking the **grantor's own privilege** does not automatically cascade down the grant chain — each node in the chain must be revoked independently
- `REVOKE ALL` only removes object-level privileges — it does **not** remove role-based or system privileges; those require `REVOKE role FROM user` separately

---

### Roles: Grouping Privileges

#### Explain It

A **role** is a named group of privileges that can be granted to users or other roles. Roles simplify administration: instead of granting 20 individual privileges to each new employee, grant them to a role once, then grant the role to each user.

Common roles: `CONNECT`, `RESOURCE`, `DBA`. Oracle 12c+ deprecates `CONNECT` and `RESOURCE` — prefer custom roles with explicit grants.

#### Prove It

```sql
-- Create a role
CREATE ROLE read_only;

-- Grant object privilege to the role
GRANT SELECT ON emp_mstr TO read_only;

-- Grant the role to a user
GRANT read_only TO hansel;

-- Hansel can now query emp_mstr without a direct GRANT
SELECT * FROM hr.emp_mstr;

-- Revoke the role from the user
REVOKE read_only FROM hansel;

-- Drop the role (removes it from all users)
DROP ROLE read_only;
```

#### Gotchas / Edge Cases

- **Roles are disabled inside definer's-rights stored procedures** — a procedure running with `AUTHID DEFINER` cannot use roles; all required privileges must be granted **directly** to the procedure owner
- `ALTER USER ... DEFAULT ROLE ALL` enables all roles at login; `DEFAULT ROLE NONE` disables all — roles must be explicitly enabled with `SET ROLE`
- Dropping a role with `DROP ROLE` **automatically revokes** it from all users who held it — no separate `REVOKE` needed

---

### Summary Table

| Command           | Purpose                                     | Scope                    |
| ----------------- | ------------------------------------------- | ------------------------ |
| `GRANT priv ON obj TO user` | Give an object privilege           | One user, one object     |
| `GRANT priv TO user`        | Give a system privilege            | One user, entire DB      |
| `GRANT role TO user`        | Give a role (group of privileges)  | One user, any privileges |
| `GRANT ... WITH GRANT OPTION` | Allow privilege to be passed on | Creates privilege chain  |
| `REVOKE priv ON obj FROM user` | Remove an object privilege    | One user, one object     |
| `REVOKE role FROM user`     | Remove a role from a user         | All privs in that role   |
| `REVOKE ALL ON obj FROM user` | Remove all object privs at once | One user, one object     |
