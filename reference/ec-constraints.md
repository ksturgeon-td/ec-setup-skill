# Elastic Compute — Object Creation Constraints

These constraints apply to ALL object creation on Elastic Compute. Violations may succeed
silently in the session but fail on the next provisioning cycle.

---

## Database Naming

- Alphanumeric characters and underscores only — no dots, special characters, or spaces
- Email-format names (e.g., `first.last@domain.com`) cause OMS repopulation failures
- Always create databases under `TD_PARENT`:
  ```sql
  CREATE DATABASE <name> FROM TD_PARENT AS PERM = 0;
  ```
- Name must be unique across all Elastic Compute instances in the site

---

## ACCESSRIGHTS — The Most Common Failure Pattern

**Always grant to ROLEs at the DATABASE level.**
Object-level and user-level grants appear to succeed but do NOT persist through reprovisioning.

```sql
-- WRONG — will not survive reprovisioning
GRANT SELECT ON mydb.sales_view TO specific_user;

-- CORRECT — persists through reprovisioning
GRANT SELECT ON mydb TO td_ce_data_user_role;
GRANT EXECUTE FUNCTION ON mydb.my_udf TO td_ce_data_user_role;
```

To find the role name to grant to, query:
```sql
SELECT RoleName FROM DBC.AllRoleRightsV WHERE DatabaseName = USER;
```
The CE-specific role (e.g., `TD_CE_FinanceAnalysts`) is what appears in GRANT statements.

---

## Authorization Objects and Cross-Instance Replication

| AUTH object location | DATALAKE replication |
|----------------------|---------------------|
| GLOBAL database | Replicates to all CE instances |
| LOCAL database | Single CE instance only |

- For shared, multi-instance access: **place AUTH objects in a GLOBAL database**
- Use `AS DEFINER TRUSTED` for team-shared credentials
- `AS INVOKER` = single-CE by definition; avoid for production datalakes
- DATALAKE and FOREIGN SERVER/TABLE objects that depend on a LOCAL AUTH object will not replicate

---

## PERM Space

New databases start with `PERM = 0` — correct for databases that only hold views and auth objects.

PERM space is required for: stored procedures, UDFs, table operators, and foreign tables.

```sql
-- Allocate space (Admin / Data Curator with Admin privileges)
CALL TD_GLOBAL.ChangeSpace('<db_name>', <bytes>, :msg);

-- Grant stored procedure and function creation rights on a database
CALL TD_GLOBAL.GRANT_USER_DB_PRIVS('<db_name>', 'TD_CREATOR');
```

Ensure `TD_PARENT` has sufficient space before allocating to a child database.

---

## PERM Tables

- PERM table data does NOT survive a provisioning cycle
- Use VOLATILE tables for temporary/intermediate data
- Persistent data belongs in object store (NOS / OTF)
- ~60 GB PERM available total after Fallback — reserved primarily for data dictionary, logs, UDFs, and stored procedures

---

## Service Accounts (Headless Users)

- ClientID users authenticating via `LOGMECH=CRED`, `BEARER`, or `SECRET` cannot be granted access to GLOBAL databases
- Known limitation — expected to be resolved Q3 2026
- Workaround: use an interactive user session for global DB object creation

---

## Known Limitations (Current as of Q3 2026)

| Limitation | Expected Fix |
|------------|-------------|
| Admin role disabled — Data Curator assumes Admin privileges | Q3 2026 |
| ACCESSRIGHTS at object/user level do not persist | Q3 2026 |
| Service accounts cannot access GLOBAL databases | Q3 2026 |
| Object repopulation delay on startup (just-in-time) | Q3 2026 |
| Dot-notation database names cause OMS repopulation failure | Q3 2026 |
| JAR/SO UDFs compiled with shared library prefix (SL) not persisted | In progress |
| DBQL/RSS/EventLog purged every 6 hours, not persisted | In progress |
| QueryGrid only connects to Teradata Active Compute in same site (≤ VCE 3.4) | Roadmap |
| Viewpoint monitors max 10 Elastic Compute instances simultaneously | Roadmap |
