# ec-setup-skill

A Claude skill for self-service data source configuration on Teradata Elastic Compute.

## What It Does

Guides users through creating the prerequisite database objects needed to access their data on
Elastic Compute — authorization objects, datalakes, foreign tables, views, and access grants —
so they can then run analytics using the [Teradata SQL analytics corpus](https://github.com/ksturgeon-td/tdsql-mcp).

## What It Doesn't Do

Generate analytics SQL. Once data sources are configured, the skill hands off to the SQL
analytics corpus (`open-table-format`, `object-store`, and related topics) for query and
analytics patterns.

## Roles Supported

| Persona | DB Role | Capability |
|---------|---------|-----------|
| Data User | `TD_ACCESS` | Discover existing data sources and query them |
| Data Curator | `TD_CREATOR` | Create auth objects, datalakes, foreign tables, views, grants |
| Admin | `TD_ADMIN` | Currently same as Data Curator (Admin role enabled post-GA) |

## Workflows

- **OTF Datalake** — Connect to Iceberg or Delta Lake via AWS Glue, Azure OneLake, Databricks Unity Catalog, GCP BigLake, Hive Metastore, or REST catalog
- **NOS Foreign Table** — Persistent access to Parquet/CSV/JSON files in S3, Azure ADLS, or GCS
- **NOS Ad-hoc Read** — Exploratory access via READ_NOS views without a persistent foreign table

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Agent instructions — decision tree, workflows, SQL templates, corpus references |
| `reference/ec-constraints.md` | EC-specific constraints and known limitations; updated as limitations are lifted |

## Dependencies

This skill references the Teradata SQL analytics corpus for SQL syntax details:
- `get_syntax_help(topic="open-table-format")` — OTF datalake DDL and query syntax
- `get_syntax_help(topic="object-store")` — NOS foreign tables and READ_NOS
- `get_syntax_help(topic="authorization-objects")` — AUTH object CREATE/REPLACE syntax
- `get_syntax_help(topic="catalog-views")` — DBC catalog views for discovery

## Elastic Compute Version

Reflects the GA release of Teradata Elastic Compute as documented in the Database User
Guide (D044535, © 2026 Teradata). Several known limitations are targeted for the Q3 2026
release — see `reference/ec-constraints.md` for the current status of each.
