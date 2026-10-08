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
SELECT RoleName FROM DBC.AllRoleRightsV WHERE DatabaseName = USER;
```

| Role found | Persona | Proceed to |
|------------|---------|-----------|
| `TD_ACCESS` | Data User | Step 2A — Discovery |
| `TD_CREATOR` or `TD_ADMIN` | Data Curator / Admin | Step 2B — Setup |
| Neither | Unknown | Ask user to contact their EC admin to confirm role assignment |

> Note: Admin role is currently disabled at the database level — Data Curator (`TD_CREATOR`)
> has assumed Admin privileges in the interim.

---

## Step 2A — Data User (TD_ACCESS): Discovery

Data Users can discover and query existing objects but cannot create databases, datalakes,
or foreign tables.

### Discover available data sources

```sql
-- Registered datalakes (OTF / Iceberg / Delta Lake)
SELECT DatalakeName, CatalogType, ObjectStoragePlatform
FROM DBC.DatalakeInfoV
ORDER BY DatalakeName;

-- If DBC.DatalakeInfoV returns no rows, enumerate TD_SERVER_DB directly
HELP DATABASE TD_SERVER_DB;
-- Rows with Kind = 'K' are foreign servers / datalakes

-- Global databases accessible across all CE instances
SELECT DatabaseName FROM DBC.DatabasesV
WHERE DatabaseName IN (
    SELECT DatabaseName FROM DBC.ChildrenV WHERE ParentName = 'TD_GLOBAL'
);

-- Views and tables the user can query in a known database
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
SELECT DatabaseName FROM DBC.DatabasesV
WHERE DatabaseName IN (
    SELECT DatabaseName FROM DBC.ChildrenV WHERE ParentName = 'TD_GLOBAL'
);
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

Use `AS DEFINER TRUSTED` for shared team access. Place it in the global database so the
datalake replicates across all CE instances.

```sql
-- Standard key/secret (AWS, GCS, NIM)
CREATE AUTHORIZATION <global_db>.<auth_name>
    AS DEFINER TRUSTED
    USER  '<access_key_or_service_account>'
    PASSWORD '<secret_key>';

-- Azure additionally requires SESSION_TOKEN = API version
CREATE AUTHORIZATION <global_db>.<auth_name>
    AS DEFINER TRUSTED
    USER  '<azure_endpoint_url>'
    PASSWORD '<api_key>'
    SESSION_TOKEN '<api_version>';

-- AWS IAM role assumption
CREATE AUTHORIZATION <global_db>.<auth_name>
    AS DEFINER TRUSTED
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

-- Grant on TD_SERVER_DB so users can query the datalake directly
GRANT SELECT ON TD_SERVER_DB TO <td_ce_data_user_role>;
```

To find the correct CE role name:
```sql
SELECT RoleName FROM DBC.AllRoleRightsV
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

**Step B1 — Create the AUTH object**

```sql
CREATE AUTHORIZATION <global_db>.<auth_name>
    AS DEFINER TRUSTED
    USER  '<access_key>'
    PASSWORD '<secret_key>';
```

**Step B2 — Create the FOREIGN TABLE**

The database hosting the foreign table needs PERM space. If the database was created with
`PERM = 0`, allocate space first:

```sql
CALL TD_GLOBAL.ChangeSpace('<db_name>', 10000000, :msg);
```

Then create the table:

```sql
CREATE MULTISET FOREIGN TABLE <db>.<table_name>,
    EXTERNAL SECURITY DEFINER TRUSTED <global_db>.<auth_name>
    USING (
        LOCATION  ('/s3/my-bucket/path/')
        STOREDAS  ('PARQUET')
    )
NO PRIMARY INDEX;
```

**Step B3 — (Optional) Create a view**

```sql
CREATE VIEW <global_db>.<view_name> AS
SELECT * FROM <db>.<table_name>;
```

**Step B4 — Grant access**

```sql
GRANT SELECT ON <db> TO <td_ce_data_user_role>;
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

```sql
CREATE AUTHORIZATION <db>.<auth_name>
    USER  '<access_key>'
    PASSWORD '<secret_key>';
```

**Step C2 — Create a view using READ_NOS**

```sql
CREATE VIEW <db>.<view_name> AS
SELECT *
FROM (
    LOCATION  = '/s3/my-bucket/path/'
    AUTHORIZATION = <db>.<auth_name>
) AS d;
```

**Step C3 — Grant access**

```sql
GRANT SELECT ON <db> TO <td_ce_data_user_role>;
```

For full READ_NOS syntax, load `get_syntax_help(topic="object-store")`.

---

## EC Constraints — Always Enforce

Load `reference/ec-constraints.md` for the full list. Key rules to apply in every workflow:

| Rule | What to do |
|------|-----------|
| Database names | Alphanumeric + underscores only; create under `TD_PARENT` |
| AUTH placement | Always in a GLOBAL database for cross-instance replication |
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
REST API to create one automatically:

```
POST /api/v1/sites/{site_id}/global-databases
{ "name": "<db_name>", "perm": 0 }
```

Until that skill is implemented, direct the user to:
**Vantage Console → Elastic Compute → [instance] → Object Metadata → + Add**
