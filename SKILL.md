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

`SELECT CURRENT_ROLE` returns "ALL" on Elastic Compute — use this query instead.
It shows both directly-granted roles and roles inherited through them:

```sql
-- Direct + nested effective roles for a specific user
-- Note: Grantee is returned lowercase — compare case-insensitively
SELECT 'direct' AS src, RoleName
FROM DBC.RoleMembersV WHERE Grantee = '<username>'
UNION ALL
SELECT 'nested', r.RoleName
FROM DBC.RoleMembersV u
JOIN DBC.RoleMembersV r ON u.RoleName = r.Grantee
WHERE u.Grantee = '<username>';
-- e.g. '<username>' = 'kevin.sturgeon@teradata.com'
```

> **Ignore `TD_SSO_*` / `SSO_EC*` roles** — these are SCIM synchronization artifacts
> from the management plane, not EC data roles. Focus on: `TD_CREATOR`, `TD_ADMIN`,
> `*_DATA_CURATOR`, `*_DATA_USER`, `TD_ACCESS`.

| Role found | Persona | Proceed to |
|------------|---------|-----------|
| `TD_ACCESS` or `*_DATA_USER` | Data User — can discover and query existing sources | Step 2A — Discovery |
| `*_DATA_CURATOR` | Data Curator — can create auth objects and tables in global databases | Step 2B — Setup |
| `TD_CREATOR` or `TD_ADMIN` | Admin — can create local databases, datalakes, call `TD_GLOBAL` procs, but **no rights inside global databases by default** | Step 2B — with rights check |
| None of the above | Unknown | Ask user to contact their EC admin to confirm role assignment |

> **Rights check for TD_CREATOR / TD_ADMIN:** Before proceeding to object creation, verify
> the user's roles have `CREATE AUTHORIZATION` (CA) or `CREATE TABLE` (CT) rights on the
> target global database. Without them, DDL will fail with Error 3524.
> ```sql
> SELECT DISTINCT RoleName, AccessRight
> FROM DBC.AllRoleRightsV
> WHERE DatabaseName = '<global_db>' AND AccessRight IN ('CA', 'CT');
> ```
> If none of the user's roles appear, advise: "You can create local objects and datalakes,
> but creating auth objects or tables in `<global_db>` requires a Data Curator role. Ask
> your EC admin."

---

## Step 2A — Data User (TD_ACCESS): Discovery

Data Users can discover and query existing objects but cannot create databases, datalakes,
or foreign tables.

### Discover available data sources

**Primary — DBC.ServerV (most reliable):**

```sql
-- Foreign servers and datalakes registered in TD_SERVER_DB
SELECT ServerName, DataBaseName, TableFormat, AuthorizationName, CatalogAuthName
FROM DBC.ServerV
ORDER BY ServerName;
-- Kind = 'K' objects in HELP DATABASE TD_SERVER_DB are datalakes/foreign servers
-- AuthorizationType may appear blank in some deployments
```

Once you have a datalake name, get its full DDL (including auth names and catalog config):
```sql
SHOW DATALAKE <datalake_name>;
```

**Secondary — DBC.DatalakeInfoV (may raise Error 3523 on some systems):**

```sql
-- If accessible, gives catalog metadata for OTF datalakes
-- Note: string values include embedded single quotes, e.g. 'glue' not glue
SELECT DatalakeName, OTFTableFormat, CatalogType, CatalogLocation,
       StorageLocation, StorageEndPoint, StorageRegion,
       UnityCatalogName, StorageAccountName
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
| Files in S3 / Azure ADLS / GCS — exploratory or single-user access, auth object required | **Workflow C — NOS Ad-hoc Read (READ_NOS view)** |
| Quick one-off query on S3 or Azure ADLS — no auth object needed, credentials inline | **Workflow D — Inline Credentials (easiest for new users)** |

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
instances. Use plain `CREATE AUTHORIZATION` (no `AS DEFINER/INVOKER`) — the EXTERNAL
SECURITY reference in Step A2 must match: plain auth → no keywords in the reference,
qualified name allowed. Mixing types raises Error 6953 or Error 3706.

```sql
-- AWS (IAM user): Access Key ID + Secret
CREATE AUTHORIZATION <global_db>.<auth_name>
    USER     '<aws_access_key_id>'
    PASSWORD '<aws_access_key_secret>';
-- AWS STS temporary creds: add SESSION_TOKEN '<aws_session_token>'

-- Azure Shared Key: Storage Account Name + Storage Account Key
CREATE AUTHORIZATION <global_db>.<auth_name>
    USER     '<storage_account_name>'
    PASSWORD '<storage_account_key>';

-- Azure SAS: Storage Account Name + Account SAS Token
CREATE AUTHORIZATION <global_db>.<auth_name>
    USER     '<storage_account_name>'
    PASSWORD '<account_sas_token>';

-- GCS (S3 interop mode): Access Key ID + Secret
CREATE AUTHORIZATION <global_db>.<auth_name>
    USER     '<gcs_access_key_id>'
    PASSWORD '<gcs_access_key_secret>';

-- GCS (native): Client Email + Private Key
CREATE AUTHORIZATION <global_db>.<auth_name>
    USER     '<service_account_client_email>'
    PASSWORD '<service_account_private_key>';

-- AWS IAM role assumption
CREATE AUTHORIZATION <global_db>.<auth_name>
    USING AUTHSERVICETYPE 'ASSUME_ROLE'
    ROLENAME '<arn:aws:iam::account-id:role/role-name>'
    EXTERNALID '<external_id>'
    DURATION_SECONDS '3600';
```

For full AUTH object syntax, load `get_syntax_help(topic="authorization-objects")`.

> **Caution:** `SHOW AUTHORIZATION` echoes the `USER` value (access key ID). Do not paste
> its output into chat, tickets, or logs.

**Step A2 — Create the DATALAKE in TD_SERVER_DB**

Plain auth → no keywords in EXTERNAL SECURITY, qualified name allowed:

```sql
CREATE DATALAKE <datalake_name>
    EXTERNAL SECURITY CATALOG <global_db>.<auth_name>,
    EXTERNAL SECURITY STORAGE <global_db>.<auth_name>
USING
    catalog_type     ('<glue|unity|hive|rest|fabric|biglake>')
    storage_location ('s3://<bucket>/<prefix>/')
    storage_region   ('<region>')
TABLE FORMAT iceberg;   -- or: deltalake  (NOT "delta")
```

> **Auth / EXTERNAL SECURITY matching rule:**
> | Auth created as | EXTERNAL SECURITY reference | Qualified name? |
> |---|---|---|
> | Plain (no `AS` clause) | `EXTERNAL SECURITY CATALOG <db>.<auth>` | Yes — verified |
> | `AS DEFINER TRUSTED` | `EXTERNAL SECURITY DEFINER TRUSTED <auth>` | No — Error 3706 |
> Mismatch raises Error 6953 (`authorization definition does not match`).

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
-- Database-level SELECT and EXECUTE — covers views, tables, and auth objects in the DB
GRANT SELECT  ON <global_db> TO <data_user_role>;
GRANT EXECUTE ON <global_db> TO <data_user_role>;
```

To find the data user role name, use the Step 1 queries — look for roles matching
`TD_ACCESS`, `*_DATA_USER`, or the site's naming convention.

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

Two verified patterns — choose one and use it consistently through B2:

**Option 1 — Plain auth (recommended for simplicity):** Auth can be in any database the
curator has access to; reference with qualified name, no keywords in EXTERNAL SECURITY.

```sql
CREATE AUTHORIZATION <db>.<auth_name>
    USER  '<access_key>'
    PASSWORD '<secret_key>';
```

**Option 2 — DEFINER TRUSTED:** Auth must be in the **same database as the table**;
reference with **unqualified name** — qualified name raises Error 3706.

```sql
CREATE AUTHORIZATION <db>.<auth_name>
    AS DEFINER TRUSTED
    USER  '<access_key>'
    PASSWORD '<secret_key>';
```

**Step B2 — Create the FOREIGN TABLE**

The database hosting the foreign table needs PERM space. `ChangeSpace` adds bytes —
check parent headroom first, read `out_msg` to confirm success:

```sql
-- Check TD_PARENT headroom (use DBC.DiskSpaceV — DatabasesV lacks CurrentPerm)
SELECT DatabaseName,
       SUM(MaxPerm)                    AS max_perm,
       SUM(CurrentPerm)                AS cur_perm,
       SUM(MaxPerm) - SUM(CurrentPerm) AS headroom
FROM DBC.DiskSpaceV
WHERE DatabaseName = 'TD_PARENT'
GROUP BY 1;

-- Add bytes — must not exceed parent headroom; run via execute_query so :msg is returned
CALL TD_GLOBAL.ChangeSpace('<db_name>', <bytes_to_add>, :msg);
-- Always read :msg — failure appears inside it (e.g. "Failure 3541 ... invalid")
-- If ChangeSpace fails, reduce <bytes_to_add> to stay within TD_PARENT headroom

-- Confirm allocation
SELECT SUM(MaxPerm) AS perm_allocated
FROM DBC.DiskSpaceV WHERE DatabaseName = '<db_name>';
```

> **Note:** `<bytes_to_add>` must not exceed TD_PARENT's available headroom. On small or
> shared instances this may be only a few hundred KB — query headroom first and size
> accordingly. Failure 3541 in `:msg` means the requested amount exceeds what's available.

Then create the table. The EXTERNAL SECURITY reference must match the auth pattern chosen
in B1. **LOCATION note:** `/s3/my-bucket/path/` treats `my-bucket` as the hostname and
fails at query time — always use the endpoint form or s3:// URI.

> **Note:** `CREATE FOREIGN TABLE` contacts the bucket at creation time (and may sample
> files for schema inference), so credentials and the storage path must be valid and
> reachable. An invalid key fails the DDL with Error 4951.

```sql
-- Option 1: plain auth, qualified reference (no keywords)
CREATE MULTISET FOREIGN TABLE <db>.<table_name>,
    EXTERNAL SECURITY <db>.<auth_name>
    USING (
        LOCATION  ('/S3/s3.amazonaws.com/<bucket>/<prefix>/')
        STOREDAS  ('PARQUET')
    )
NO PRIMARY INDEX;

-- Option 2: DEFINER TRUSTED auth, unqualified reference (auth in same DB)
CREATE MULTISET FOREIGN TABLE <db>.<table_name>,
    EXTERNAL SECURITY DEFINER TRUSTED <auth_name>
    USING (
        LOCATION  ('/S3/s3.amazonaws.com/<bucket>/<prefix>/')
        STOREDAS  ('PARQUET')
    )
NO PRIMARY INDEX;

-- Valid LOCATION formats:
--   /S3/s3.amazonaws.com/bucket/prefix/      (slash-path, uppercase S3)
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
-- Database-level grants — persist through reprovisioning
GRANT SELECT  ON <db> TO <data_user_role>;
GRANT EXECUTE ON <db> TO <data_user_role>;
```

Use the Step 1 queries to find the correct data user role name.

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
supported in view definitions. **Note:** `CREATE VIEW` over `READ_NOS` contacts the
bucket at creation time, so credentials and the storage path must be valid and reachable.

```sql
CREATE VIEW <db>.<view_name> AS
SELECT *
FROM READ_NOS (
    USING
        LOCATION      ('/S3/s3.amazonaws.com/<bucket>/<prefix>/')
        AUTHORIZATION (<db>.<auth_name>)
        STOREDAS      ('PARQUET')
) AS d;
-- Valid LOCATION formats:
--   /S3/s3.amazonaws.com/bucket/prefix/   (slash-path)
--   s3://bucket/prefix/                    (URI form)
```

**Step C3 — Grant access**

```sql
GRANT SELECT  ON <db> TO <data_user_role>;
GRANT EXECUTE ON <db> TO <data_user_role>;
```

Use the Step 1 queries to find the correct data user role name.

For full READ_NOS syntax, load `get_syntax_help(topic="object-store")`.

---

### Workflow D — Inline Credentials (No Auth Object Required)

**Best for:** New users doing a quick one-off exploration of S3 or Azure ADLS data without
setting up any auth objects first. **Supported platforms: S3 and Azure ADLSv2 only.**

Instead of creating an authorization object, pass credentials as a JSON string directly in
the `AUTHORIZATION` parameter. No `CREATE AUTHORIZATION` or `GRANT EXECUTE` needed.

**Collect from the user:**
1. Storage location (S3 URI or Azure ADLS path)
2. Credentials (Access Key ID + Secret; Session Token if using STS)

**Step D1 — Run an ad-hoc query**

Credentials are embedded as a JSON string. Supported keys: `Access_ID`, `Access_Key`,
`Session_Token` (optional — STS temporary creds only).

```sql
-- Explicit READ_NOS form (recommended)
SELECT TOP 10 *
FROM READ_NOS (
    USING
        LOCATION      ('/S3/s3.amazonaws.com/<bucket>/<prefix>/')
        AUTHORIZATION ('{"Access_ID":"<access_key_id>","Access_Key":"<secret_key>"}')
        RETURNTYPE    ('NOSREAD_KEYS')
) AS d;

-- Azure ADLS (same JSON format — Storage Account Name + Key)
SELECT TOP 10 *
FROM READ_NOS (
    USING
        LOCATION      ('/AZ/<storage_account>.blob.core.windows.net/<container>/<prefix>/')
        AUTHORIZATION ('{"Access_ID":"<storage_account_name>","Access_Key":"<storage_account_key>"}')
        RETURNTYPE    ('NOSREAD_KEYS')
) AS d;

-- Implicit shorthand form (also valid — recognize it but prefer explicit above)
SELECT TOP 10 * FROM (
    LOCATION      = '/S3/s3.amazonaws.com/<bucket>/<prefix>/'
    AUTHORIZATION = '{"Access_ID":"<access_key_id>","Access_Key":"<secret_key>"}'
    RETURNTYPE    = 'NOSREAD_KEYS'
) AS d;
```

**Step D2 — (Optional) Wrap in a view**

```sql
CREATE VIEW <db>.<view_name> AS
SELECT *
FROM READ_NOS (
    USING
        LOCATION      ('/S3/s3.amazonaws.com/<bucket>/<prefix>/')
        AUTHORIZATION ('{"Access_ID":"<access_key_id>","Access_Key":"<secret_key>"}')
        STOREDAS      ('PARQUET')
) AS d;
-- Note: credentials are stored in the view definition — anyone with SELECT on
-- the view can read them. Use a proper auth object (Workflow C) for shared views.
```

> **When to upgrade to Workflow C:** Once a data source is used by more than one person,
> or when you want credentials out of the query text, create an auth object instead.
> Inline credentials are visible in query logs and view DDL.

For full READ_NOS syntax and additional examples, load `get_syntax_help(topic="object-store")`.

---

## EC Constraints — Always Enforce

Load `reference/ec-constraints.md` for the full list. Key rules to apply in every workflow:

| Rule | What to do |
|------|-----------|
| Database names | Alphanumeric + underscores only; create under `TD_PARENT` — applies to explicitly created databases; system user accounts may use email-format names |
| AUTH / EXTERNAL SECURITY matching | Plain auth → `EXTERNAL SECURITY <db>.<auth>` (no keywords, qualified OK); DEFINER TRUSTED → `EXTERNAL SECURITY DEFINER TRUSTED <auth>` (unqualified, same DB as object). Mismatch → Error 3706 or 6953 |
| AUTH placement (DATALAKE) | Must be in a GLOBAL database — datalakes replicate globally; plain auth verified |
| AUTH placement (NOS foreign tables) | Same DB as the table for DEFINER TRUSTED pattern; any accessible DB for plain auth pattern |
| AUTH placement (READ_NOS views) | Any accessible DB; plain auth |
| ACCESSRIGHTS | Grant to ROLE at DATABASE level only (`GRANT SELECT/EXECUTE ON <db> TO <role>`) — object-level grants do not survive reprovisioning |
| PERM space | Default `PERM = 0`; allocate only for foreign tables, UDFs, stored procs. `ChangeSpace` adds bytes — always read `out_msg` and re-query PermSpace to confirm |
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
