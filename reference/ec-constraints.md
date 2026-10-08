# Elastic Compute — Object Creation Constraints

These constraints apply to ALL object creation on Elastic Compute. Violations may succeed
silently in the session but fail on the next provisioning cycle.

---

## Database Naming

- Alphanumeric characters and underscores only — no dots, special characters, or spaces
- This restriction applies to databases explicitly created under `TD_PARENT` — system-managed
  user accounts may have email-style names by design; do not try to rename those
- Email-format names in explicitly created databases (e.g., `first.last_domain_com`) cause
  OMS repopulation failures
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

To find the role name to grant to, use the Step 1 queries in the skill. Role naming
conventions vary by deployment — look for patterns like `TD_ACCESS`, `*_DATA_USER`, or
site-specific names. Do not assume a `TD_CE_*` prefix exists on every system.

---

## Authorization Objects and Cross-Instance Replication

AUTH placement rules differ by object type:

| Object type | AUTH requirement | EXTERNAL SECURITY reference |
|-------------|-----------------|----------------------------|
| DATALAKE | AUTH in GLOBAL database, plain (no `AS` clause) | `EXTERNAL SECURITY CATALOG <db>.<auth>` — qualified name, no keywords |
| FOREIGN TABLE (plain) | AUTH in any accessible DB, plain | `EXTERNAL SECURITY <db>.<auth>` — qualified name, no keywords |
| FOREIGN TABLE (DEFINER) | AUTH in **same DB as the table**, `AS DEFINER TRUSTED` | `EXTERNAL SECURITY DEFINER TRUSTED <auth>` — unqualified only; Error 3706 if qualified |
| READ_NOS view | AUTH in any accessible DB, plain | `AUTHORIZATION (<db>.<auth_name>)` — qualified, no keywords |

**Matching rule:** The keywords in `EXTERNAL SECURITY` must match how the auth object was
created. A mismatch raises Error 6953 (`authorization definition does not match`) or
Error 3706 (qualified name with DEFINER TRUSTED).

Additional rules:
- `AS INVOKER` auth = single-CE by definition; suitable for READ_NOS, not for production datalakes
- AUTH objects that depend on LOCAL databases will not replicate with datalakes

**INVOKER note:** When using `AS INVOKER`, the auth object can generally be created by an
Admin in any database, or by a Curator in databases they own. Exact privilege requirements
may vary — verify with `HELP DATABASE <db>` to confirm CREATE AUTHORIZATION rights.

---

## PERM Space

New databases start with `PERM = 0` — correct for databases that only hold views and auth objects.

PERM space is required for: stored procedures, UDFs, table operators, and foreign tables.

```sql
-- Check parent headroom before allocating
SELECT DatabaseName, PermSpace, CurrentPerm
FROM DBC.DatabasesV WHERE DatabaseName = 'TD_PARENT';

-- Add bytes to the target database (2nd argument is bytes TO ADD, not total)
CALL TD_GLOBAL.ChangeSpace('<db_name>', <bytes_to_add>, :msg);
-- Always read :msg — failure appears as text in the output (e.g. Failure 3541)
-- Then re-query to confirm:
SELECT DatabaseName, PermSpace FROM DBC.DatabasesV WHERE DatabaseName = '<db_name>';

-- Grant stored procedure and function creation rights (LOCAL databases only)
-- GRANT_USER_DB_PRIVS works on TD_PARENT-owned local DBs; does NOT apply to
-- global databases or authorization/table rights
CALL TD_GLOBAL.GRANT_USER_DB_PRIVS('<db_name>', 'TD_CREATOR');
```

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
