# Security Management Using SQL

## Granting and Revoking Permissions

Oracle provides extensive security features to safeguard information stored in its tables from unauthorized viewing and damage. Depending on a user's status and responsibility, appropriate rights (privileges) can be assigned by the DBA. The rights that allow the use of some or all of Oracle's resources are called **Privileges**.

Objects created by a user are **owned and controlled** by that user. To access another user's objects, the owner must grant permissions. This is called **Granting of Privileges**. Privileges can be revoked by the owner, called **Revoking of Privileges**.

---
## Granting Privileges Using the GRANT Statement

The GRANT statement provides access to database objects (tables, views, sequences, etc.).

**Syntax:**
```sql
GRANT <Object Privileges>
ON <ObjectName>
TO <UserName>
[WITH GRANT OPTION];
```

### Object Privileges
- **ALTER**: Change table definition (ALTER TABLE)
- **DELETE**: Remove records (DELETE)
- **INDEX**: Create index (CREATE INDEX)
- **INSERT**: Add records (INSERT)
- **SELECT**: Query table (SELECT)
- **UPDATE**: Modify records (UPDATE)

**WITH GRANT OPTION**: Allows the grantee to grant privileges to other users.

---
## Examples
- Grant all permissions:
  ```sql
  GRANT ALL ON EMP_MSTR TO Sharanam;
  ```
- Grant select and update:
  ```sql
  GRANT SELECT, UPDATE ON CUST_MSTR TO Hansel;
  ```
- Grant with grant option:
  ```sql
  GRANT ALL ON ACCT_MSTR TO Ivan WITH GRANT OPTION;
  ```

---
## Referencing Another User's Table
- Access another user's table by prefixing with owner name:
  ```sql
  SELECT * FROM Sharanam.FD_MSTR;
  ```

---
## Granting Privileges When Grantee Has GRANT Privilege
- Example:
  ```sql
  GRANT SELECT ON Vaishali.TRANS_MSTR TO Chhaya;
  ```

---
## Revoking Privileges Using the REVOKE Statement

Privileges can be revoked using the REVOKE command.

**Syntax:**
```sql
REVOKE <Object Privileges>
ON <ObjectName>
FROM <UserName>;
```

- REVOKE cannot revoke privileges granted by the operating system.

---
## Examples
- Revoke delete privilege:
  ```sql
  REVOKE DELETE ON NOMINEE_MSTR FROM Anil;
  ```
- Revoke all privileges:
  ```sql
  REVOKE ALL ON NOMINEE_MSTR FROM Anil;
  ```
- Revoke select privilege:
  ```sql
  REVOKE SELECT ON Alex.FDSLAB_MSTR FROM Rocky;
  ```

---
## Self Review Questions
1. The rights that allow the use of some or all of Oracle's resources on the Server are called _________.
2. Objects that are created by a user are owned and controlled by _________.
3. The ________ statement provides various types of access to database objects.
4. ________ privilege allows the grantee to remove records from the table.
5. ________ privilege allows the grantee to query the table.
6. The ________ allows the grantee to in turn grant object privileges to other users.

---
**Tip:** Always create users before granting permissions. Use GRANT and REVOKE to manage access securely.
