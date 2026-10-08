---
name: elastic-compute-setup
description: Self-service data source setup on Teradata Elastic Compute. Guides users through role detection, auth objects, datalakes, foreign tables, views, and access grants based on their EC role (TD_ACCESS, TD_CREATOR, TD_ADMIN).
---

# Elastic Compute Setup Skill

## Purpose

Guide users through self-service data source configuration on Teradata Elastic Compute (EC).
This skill handles prerequisite object creation — authorization objects, datalakes, foreign
tables, views, and access grants — so users can then query and analyze data using the
Teradata SQL analytics corpus.

This skill does NOT generate analytics SQL. Once data sources are configured, delegate to
`get_syntax_help(topic="open-table-format")`, `get_syntax_help(topic="object-store")`, or
the broader corpus for query and analytics patterns.

---

## Activation

Invoke this skill when a user on an Elastic Compute instance asks to:
- Connect to their data (Iceberg, Delta Lake, S3, Azure ADLS, GCS, etc.)
- Set up access to an OTF catalog or object store
- Create database objects on Elastic Compute
- Understand what data sources are available to them
- Grant other users access to data sources

---

## Step 1 — Identify the User's Role

`SELECT CURRENT_ROLE` returns "ALL" on Elastic Compute — use this query instead:

```sql
SELECT DISTINCT RoleName FROM DBC.AllRoleRightsV WHERE DatabaseName = USER;
```

| Role found | Persona | Proceed to |
|------------|---------|-----------|
| `TD_ACCESS` | Data User | Step 2A — Discovery |
| `TD_CREATOR` | Data Curator | Step 2B — Setup |
| `TD_ADMIN` | Admin | Step 2B — Setup (treat as Curator; Admin role may not be active) |
| None of the above | Unknown | Ask user to contact their EC admin to confirm role assignment |

> Note: `TD_ADMIN` may or may not be present in a given deployment. When it is, treat the
> user as a Data Curator — Admin-specific capabilities are not yet enabled at the DB level.

---

## Step 2A — Data User (TD_ACCESS): Discovery

Data Users can discover and query existing objects but cannot create databases, datalakes,
or foreign tables.

### Discover available data sources

**Primary — DBC.ServerV (most reliable):**

```sql
-- Foreign servers and datalakes registered in TD_SERVER_DB
SELECT ServerName, DataBaseName, AuthorizationName, TableFormat
FROM DBC.ServerV
ORDER BY ServerName;
-- Kind = 'K' objects in HELP DATABASE TD_SERVER_DB are datalakes/foreign servers
```

**Secondary — DBC.DatalakeInfoV (may raise Error 3523 on some systems):**

```sql
-- If accessible, gives catalog metadata for OTF datalakes
SELECT DatalakeName, OTFTableFormat, CatalogType, CatalogLocation,
       StorageLocation, StorageEndPoint, StorageRegion
FROM DBC.DatalakeInfoV
ORDER BY DatalakeName;
-- If Error 3523 or empty result, fall back to DBC.ServerV + SHOW DATALAKE <name>
```

**Global databases accessible across all CE instances:**

```sql
SELECT Child AS DatabaseName FROM DBC.ChildrenV WHERE Parent = 'TD_GLOBAL';
```

**Views and tables the user can query in a known global database:**

```sql
SELECT TableName, TableKind
FROM DBC.TablesV
WHERE DatabaseName = '<global_db>'
ORDER BY TableKind, TableName;
```

For the full OTF discovery workflow (HELP DATALAKE, HELP DATABASE, HELP TABLE),
load `get_syntax_help(topic="catalog-views")`.

Once sources are located, query OTF data using three-part notation (`datalake.db.table`) —
load `get_syntax_help(topic="open-table-format")` for query syntax.

**If no datalakes or global databases exist:**
> "No data sources have been configured yet on this Elastic Compute instance. Contact your
> Data Curator or EC Admin to set up access."

---

## Step 2B — Data Curator / Admin (TD_CREATOR): Setup

### Pre-flight: Global Database Check

Before any object creation, verify at least one global database exists:

```sql
SELECT Child AS DatabaseName FROM DBC.ChildrenV WHERE Parent = 'TD_GLOBAL';
```

**If no global databases exist**, stop and respond:
> "A global database is required before creating shared authorization objects or datalakes.
> Global databases are created by Org Admins or Site Admins via Vantage Console →
> Object Metadata. Please contact your admin to create one first.
>
> [Future: this will trigger the EC global database creation REST API automatically.]"

**Do not proceed with AUTH or DATALAKE creation until a global database exists.**
AUTH objects in LOCAL databases will not replicate — datalakes that depend on them will
exist only on the current CE instance.

---

### Choose a Workflow

Ask the user what they are connecting to:

| Data source | Workflow |
|-------------|---------|
| Iceberg or Delta Lake catalog (AWS Glue, Azure OneLake, Databricks Unity Catalog, GCP BigLake, Hive Metastore, Polaris, Gravitino, etc.) | **Workflow A — OTF Datalake** |
| Files in S3 / Azure ADLS / GCS without an Iceberg catalog (Parquet, CSV, JSON) — persistent, shared access | **Workflow B — NOS Foreign Table** |
| Files in S3 / Azure ADLS / GCS — exploratory or single-user access | **Workflow C — NOS Ad-hoc Read (READ_NOS view)** |

---

### Workflow A — OTF Datalake (Iceberg / Delta Lake)

**Collect from the user:**
1. Which global database will hold the AUTH object?
2. Catalog type: `glue` / `unity` / `hive` / `rest` / `fabric` / `biglake`
3. Catalog URL or AWS region
4. Cloud storage platform: S3 / Azure / GCS
5. Credentials (access key + secret key, or AWS IAM role ARN + external ID)
6. Datalake name — alphanumeric and underscores only; becomes the top-level name in three-part queries (`<datalake>.<db>.<table>`)

**Step A1 — Create the AUTH object in the global database**

Place the AUTH object in the global database so the datalake replicates across all CE
instances. For DATALAKE objects, use plain `CREATE AUTHORIZATION` (no DEFINER/INVOKER) —
`AS DEFINER TRUSTED` with DATALAKE is untested and may not be supported.

```sql
-- Standard key/secret (AWS, GCS, NIM)
CREATE AUTHORIZATION <global_db>.<auth_name>
    USER  '<access_key_or_service_account>'
    PASSWORD '<secret_key>';
-- AWS only: add SESSION_TOKEN for STS-issued temporary credentials
--   SESSION_TOKEN '<aws_session_token>'

-- Azure additionally requires SESSION_TOKEN = API version
CREATE AUTHORIZATION <global_db>.<auth_name>
    USER  '<azure_endpoint_url>'
    PASSWORD '<api_key>'
    SESSION_TOKEN '<api_version>';

-- AWS IAM role assumption
CREATE AUTHORIZATION <global_db>.<auth_name>
    USING AUTHSERVICETYPE 'ASSUME_ROLE'
    ROLENAME '<arn:aws:iam::account-id:role/role-name>'
    EXTERNALID '<external_id>'
    DURATION_SECONDS '3600';
```

For full AUTH object syntax, load `get_syntax_help(topic="authorization-objects")`.

**Step A2 — Create the DATALAKE in TD_SERVER_DB**

```sql
CREATE DATALAKE <datalake_name>
    EXTERNAL SECURITY DEFINER TRUSTED CATALOG <global_db>.<auth_name>,
    EXTERNAL SECURITY DEFINER TRUSTED STORAGE <global_db>.<auth_name>
USING catalog_type('<glue|unity|hive|rest|fabric|biglake>')
      ... TABLE FORMAT <ICEBERG|DELTA>;
```

For full `CREATE DATALAKE` syntax including catalog-specific parameters (catalog URL,
storage location, table format options), load `get_syntax_help(topic="open-table-format")`.

**Step A3 — (Optional) Create views in the global database**

Views expose specific tables or filtered slices to Data Users without exposing the full datalake.

```sql
CREATE VIEW <global_db>.<view_name> AS
SELECT * FROM <datalake_name>.<otf_database>.<otf_table>
WHERE <filter>;
```

**Step A4 — Grant access to the Data User role at the DATABASE level**

```sql
-- Grant on the global database (covers auth objects and views)
GRANT SELECT ON <global_db> TO <td_ce_data_user_role>;

-- Grant EXECUTE on the auth object so users can use it
GRANT EXECUTE ON <global_db>.<auth_name> TO <td_ce_data_user_role>;

-- Grant on TD_SERVER_DB so users can query datalake tables directly
-- (DATALAKE objects live in TD_SERVER_DB — users need SELECT here to reach them)
GRANT SELECT ON TD_SERVER_DB TO <td_ce_data_user_role>;
```

To find the correct CE Data User role name:
```sql
SELECT DISTINCT RoleName FROM DBC.AllRoleRightsV
WHERE DatabaseName = USER AND RoleName LIKE 'TD_CE_%';
```

> See `reference/ec-constraints.md` for critical rules on ACCESSRIGHTS persistence.

---

### Workflow B — NOS Foreign Table (Persistent, Shared)

Best for file-based object store data where multiple users need persistent, performant access.

**Collect from the user:**
1. Which global or local database will hold the table and AUTH object?
2. Cloud storage location (S3 URI / Azure container URL / GCS path)
3. File format: `PARQUET` / `CSV` / `JSON`
4. Credentials
5. Table name (alphanumeric and underscores only)

**Step B1 — Create the AUTH object in the same database as the table**

For FOREIGN TABLE, use `AS DEFINER TRUSTED` and place the auth object in the **same
database as the table**. Reference it with an unqualified name in the EXTERNAL SECURITY
clause — using a qualified name (db.auth) raises Error 3706.

```sql
CREATE AUTHORIZATION <db>.<auth_name>
    AS DEFINER TRUSTED
    USER  '<access_key>'
    PASSWORD '<secret_key>';
```

**Step B2 — Create the FOREIGN TABLE**

The database hosting the foreign table needs PERM space. First verify `TD_PARENT` has
sufficient free space, then allocate to the child database:

```sql
-- Check TD_PARENT free space (optional but recommended)
SELECT PermSpace, CurrentPerm, MaxPerm
FROM DBC.AllSpaceV WHERE DatabaseName = 'TD_PARENT';

-- Allocate space to the target database
CALL TD_GLOBAL.ChangeSpace('<db_name>', 10000000, :msg);

-- Verify allocation succeeded
SELECT CurrentPerm, MaxPerm FROM DBC.AllSpaceV WHERE DatabaseName = '<db_name>';
```

Then create the table. Use a valid LOCATION URI — either slash-path or s3:// form:

```sql
CREATE MULTISET FOREIGN TABLE <db>.<table_name>,
    EXTERNAL SECURITY DEFINER TRUSTED <auth_name>
    USING (
        LOCATION  ('/S3/s3.amazonaws.com/my-bucket/path/')
        STOREDAS  ('PARQUET')
    )
NO PRIMARY INDEX;
-- Valid LOCATION formats:
--   /S3/s3.amazonaws.com/bucket/prefix/      (slash-path, note uppercase S3)
--   s3://bucket/prefix/                       (URI form)
--   /AZ/storageacct.blob.core.windows.net/container/prefix/
--   /GS/storage.googleapis.com/bucket/prefix/
```

**Step B3 — (Optional) Create a view**

```sql
CREATE VIEW <global_db>.<view_name> AS
SELECT * FROM <db>.<table_name>;
```

**Step B4 — Grant access**

```sql
GRANT SELECT ON <db> TO <td_ce_data_user_role>;
GRANT EXECUTE ON <db>.<auth_name> TO <td_ce_data_user_role>;
```

To find the correct CE Data User role name:
```sql
SELECT DISTINCT RoleName FROM DBC.AllRoleRightsV
WHERE DatabaseName = USER AND RoleName LIKE 'TD_CE_%';
```

For full NOS syntax (PATHPATTERN, schema inference, SNAPSHOT_LOCATION, import workflow),
load `get_syntax_help(topic="object-store")`.

---

### Workflow C — NOS Ad-hoc Read (READ_NOS View)

Best for exploratory access by a single user or team, or when a persistent foreign table is
not needed. The AUTH object can be in any database the curator has access to.

**Collect from the user:**
1. Target database for the view and AUTH object
2. Cloud storage location
3. Credentials
4. View name

**Step C1 — Create the AUTH object**

Plain auth (no DEFINER/INVOKER) can be placed in any database the curator has access to.

```sql
CREATE AUTHORIZATION <db>.<auth_name>
    USER  '<access_key>'
    PASSWORD '<secret_key>';
```

**Step C2 — Create a view using READ_NOS**

Use the explicit `READ_NOS(USING ...)` table operator form — inline shorthand is not
supported in view definitions:

```sql
CREATE VIEW <db>.<view_name> AS
SELECT *
FROM READ_NOS (
    USING
        LOCATION ('/S3/s3.amazonaws.com/my-bucket/path/')
        AUTHORIZATION (COLUMN <db>.<auth_name>)
        STOREDAS ('PARQUET')
) AS d;
-- Valid LOCATION formats:
--   /S3/s3.amazonaws.com/bucket/prefix/   (slash-path)
--   s3://bucket/prefix/                    (URI form)
```

**Step C3 — Grant access**

```sql
GRANT SELECT ON <db> TO <td_ce_data_user_role>;
GRANT EXECUTE ON <db>.<auth_name> TO <td_ce_data_user_role>;
```

For full READ_NOS syntax, load `get_syntax_help(topic="object-store")`.

---

## EC Constraints — Always Enforce

Load `reference/ec-constraints.md` for the full list. Key rules to apply in every workflow:

| Rule | What to do |
|------|-----------|
| Database names | Alphanumeric + underscores only (no dots/spaces); create under `TD_PARENT` — this restriction applies to explicitly created databases, not system-managed user accounts |
| AUTH placement (DATALAKE) | AUTH objects for DATALAKE objects must be in a GLOBAL database — datalakes always replicate globally, so their auth objects must too |
| AUTH placement (NOS foreign tables) | Auth can be in the same database as the foreign table (local or global); must be DEFINER TRUSTED with unqualified name |
| AUTH placement (READ_NOS views) | Auth can be in any database the curator has access to; plain auth (no DEFINER/INVOKER) |
| ACCESSRIGHTS | Grant to ROLE at DATABASE level only — never to users or individual objects |
| PERM space | Default `PERM = 0`; allocate only when storing UDFs, stored procs, or foreign tables |
| PERM tables | Never rely on them for persistent data — use OTF or NOS |

---

## Corpus Cross-References

| Need | Load this topic |
|------|----------------|
| OTF datalake DDL, HELP commands, time travel, DML | `open-table-format` |
| NOS foreign tables, READ_NOS, WRITE_NOS, PATHPATTERN | `object-store` |
| AUTH object CREATE/REPLACE syntax, DEFINER/INVOKER/TRUSTED | `authorization-objects` |
| DBC catalog views, TD_SERVER_DB discovery | `catalog-views` |

---

## Future Integration (Stub)

When a global database is required but none exists, this skill will trigger the EC Admin
REST API to create one automatically. The specific endpoint path is not yet confirmed —
do not generate or guess it.

Until that API integration is implemented, direct the user to:
**Vantage Console → Elastic Compute → [instance] → Object Metadata → + Add**
