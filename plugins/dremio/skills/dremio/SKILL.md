---
name: dremio
description: Working with Dremio Cloud, including ingesting, transforming, and querying data. Use when the user asks about data, tables, schemas, SQL queries, datasets, analytics, or anything involving structured data.
user-invocable: false
---

# Dremio Cloud

A project contains **namespaces** (managed Iceberg storage, full DDL/DML) and **sources** (connections to external systems). Sources can be object storage (S3, ADLS), databases (Oracle, Postgres, MongoDB), or lakehouse catalogs (Unity Catalog, Glue). Lakehouse sources with Iceberg tables may support read/write. Tables in SQL: `"Namespace"."table"` or `"Namespace"."Folder"."table"`.

## REST API

Use the REST API for all Dremio interactions. Base URL: `https://api.dremio.cloud/v0/projects/${DREMIO_PROJECT_ID}`.

**Auth setup:** `DREMIO_PAT` and `DREMIO_PROJECT_ID` must be set. The `.env` file should use `export` and single quotes (PATs often contain `+`, `/`, `=` which can be mangled by the shell):

```
export DREMIO_PAT='your_pat_here'
export DREMIO_PROJECT_ID='your_project_id_here'
```

If the `.env` doesn't exist, ask the user to create one with their PAT and project ID.

**zsh + special characters:** PATs often contain `+`, `/`, `=` which zsh can silently corrupt. Either regenerate the PAT until you get one without `+`, or wrap REST API calls in `bash -c '.  /path/to/.env && curl ...'`.

Use `jq` (not python) to parse JSON responses.

### Search API

Find tables, views, and other entities by keyword:

```bash
curl -s -X POST "https://api.dremio.cloud/v0/projects/${DREMIO_PROJECT_ID}/search" \
  -H "Authorization: Bearer $DREMIO_PAT" -H "Content-Type: application/json" \
  -d '{"query": "search terms", "filter": "category in [\"TABLE\", \"VIEW\"]", "maxResults": 10}'
```

Filter categories: `TABLE`, `VIEW`, `FOLDER`, `SPACE`, `SOURCE`, `REFLECTION`, `UDF`, `SCRIPT`, `JOB`. Results include paths, columns, wiki descriptions, and labels.

## Workflow

1. **Find tables.** Use the Search API with keywords. Fallback: `INFORMATION_SCHEMA."TABLES"` with `TABLE_TYPE != 'SYSTEM_TABLE'`.
2. **Understand schema.** Search API returns columns inline. For more detail, query `INFORMATION_SCHEMA."COLUMNS"`.
3. **Query.** Use the SQL API for all queries (SELECT, DDL, DML).
4. **Present results** as markdown tables. Summarize findings. Suggest follow-ups.

## Data Ingestion

- **CTAS** (`CREATE TABLE AS SELECT`) — one step, but only works on data already queryable in Dremio.
- **COPY INTO** — loads external files (CSV, JSON, Parquet) into a namespace table. **Table must exist first.**

For new data in S3: prefer CREATE TABLE + COPY INTO over promoting raw files (Iceberg is more efficient).

```sql
-- CTAS (from existing data)
CREATE TABLE Namespace.summary AS SELECT state, SUM(amount) FROM Namespace.raw GROUP BY state;

-- COPY INTO (from external files — two steps)
CREATE TABLE Namespace.my_table (id INT, name VARCHAR, created_at TIMESTAMP);
COPY INTO Namespace.my_table FROM '@source/path/to/data/' FILE_FORMAT 'parquet';
```

## Folder Management

Folders organize tables and views within namespaces. Use the REST API.

```sql
CREATE FOLDER Namespace.staging
CREATE FOLDER Namespace.staging.temp          -- nested
CREATE FOLDER IF NOT EXISTS Namespace.staging -- idempotent create
DROP FOLDER Namespace.staging.temp            -- must drop children first
DROP FOLDER Namespace.staging                 -- cannot drop non-empty folders
```

`DROP FOLDER IF EXISTS` is **not supported** — omit `IF EXISTS`.

## dbt-dremio

### Profile

```yaml
# profiles.yml
my_project:
  target: dev
  outputs:
    dev:
      type: dremio
      threads: 4
      cloud_host: api.dremio.cloud       # NOT software_host
      cloud_project_id: "{{ env_var('DREMIO_PROJECT_ID') }}"
      use_ssl: true
      user: user@example.com             # required for Cloud
      pat: "{{ env_var('DREMIO_PAT') }}"
      dremio_space: MyNamespace           # database = namespace
      dremio_space_folder: no_schema      # schema = omitted (root)
      object_storage_source: MyNamespace  # where tables materialize
      object_storage_path: no_schema
```

### Database/schema mapping

dbt-dremio maps `dremio_space` → database, `dremio_space_folder` → schema. The sentinel `no_schema` omits the schema component. **There is no equivalent sentinel for the database position** — `no_schema`, `no_database`, and `""` all emit as literal strings. The namespace must always go in `dremio_space` (database).

In `sources.yml`, use `schema: no_schema` to avoid doubling. If `dremio_space: Medicaid`, do NOT set `schema: Medicaid` — that produces `"Medicaid"."Medicaid"."table"`.

### Running

Requires Python <=3.13 (3.14 breaks dbt-dremio). Use a venv:

```bash
cd <dbt-project-dir>
source .env              # sets DREMIO_PAT and DREMIO_PROJECT_ID
source .venv/bin/activate
dbt compile              # check SQL in target/compiled/
dbt run                  # materialize models
dbt test                 # run data tests
```

Model naming: `stg_` (staging views), `int_` (intermediate views), `mart_` (materialized tables). All flat in the namespace root.

## Dremio SQL Reference

### Reserved Keywords (must quote as aliases)

`day`, `month`, `year`, `hour`, `minute`, `second`, `count`, `value`, `rows`, `timestamp`, `date`, `time`, `covar_pop`, `table`, `order`, `group`, `key`, `type`, `user`, `name`, `status`

Use `AS "day"` not `AS day`.

### General Rules

- Fully qualify table names: `"Namespace"."table"` or `"Namespace"."Folder"."table"`
- LIMIT results to 100 by default unless aggregating
- Handle NULLs with COALESCE or IS NOT NULL
- Use aliases for computed columns

### Function Syntax (Dremio-specific)

| Function | Correct Syntax | Common Mistake |
|:---------|:---------------|:---------------|
| ILIKE | `ILIKE(col, pattern)` | `col ILIKE pattern` |
| PERCENTILE_CONT | `PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY col)` | Other variants |
| DATEDIFF | `DATEDIFF(start, end)` — 2 args, returns days | Unit argument |
| TIMESTAMPDIFF | `TIMESTAMPDIFF(unit, start, end)` | DATEDIFF with units |
| TIMESTAMPADD | `TIMESTAMPADD(unit, count, ts)` | |
| EXTRACT | `EXTRACT(DAY FROM ts)` | |
| JSON access | `CONVERT_FROM(col, 'JSON')."field"` | `JSON_EXTRACT_SCALAR`, `->>`  |
| STRUCT access | `struct_col."field"` | `col['field']` |
| ANY_VALUE | Not supported — use `MIN(col)` or `MAX(col)` | `ANY_VALUE(col)` |
| FLATTEN | `FLATTEN(col)` or `CROSS JOIN UNNEST(col)` | `LATERAL FLATTEN` |
| ARRAY_AGG | Not supported for STRUCT/MAP/LIST; 32KB limit | |
| Intervals | `INTERVAL '1' HOUR` (unquoted unit) | Quoting the unit |
| Large intervals | `TIMESTAMPADD(DAY, -365, CURRENT_DATE)` | `INTERVAL '365' DAY` |

## Unstructured Data & AI_GENERATE

Use `AI_GENERATE` + `LIST_FILES` to extract structured data from images, PDFs, and documents in object storage.

### Basic pattern

```sql
CREATE TABLE Namespace.Results AS
WITH
  files AS (
    SELECT file, file['path'] AS file_path
    FROM TABLE(LIST_FILES('@source/path/to/files/'))
  ),
  extracted AS (
    SELECT file_path,
      AI_GENERATE('model-name',   -- e.g. 'FieldOrg', 'claude.claude-sonnet-4-6'
        ('Your extraction prompt.', file)
        WITH SCHEMA ROW(field1 VARCHAR, field2 DOUBLE)
      ) AS data
    FROM files
  )
SELECT data['field1'] AS field1, data['field2'] AS field2, file_path
FROM extracted;
```

### Variable-length extraction (JSON array → FLATTEN)

For documents with variable numbers of items (e.g., invoice line items):

1. Extract with `line_items_json VARCHAR` in the schema — prompt the LLM to return a JSON array
2. Parse and flatten:

```sql
CREATE VIEW Namespace.LineItems AS
WITH parsed AS (
  SELECT *, TRY_CONVERT_FROM(
    line_items_json AS ARRAY(ROW(field_a VARCHAR, amount DOUBLE))
  ) AS items
  FROM Namespace.Documents
),
flattened AS (
  SELECT doc_id, FLATTEN(items) AS item FROM parsed
)
SELECT doc_id, item['field_a'] AS field_a, item['amount'] AS amount
FROM flattened;
```

### Key points
- Pass `file` column from `LIST_FILES` as second tuple element for images/PDFs
- Access fields: `data['field_name']`
- **Use CREATE TABLE** (not view) to materialize AI results — avoids re-running LLM on each query
- For variable-length data: extract as VARCHAR JSON array, then `TRY_CONVERT_FROM` + `FLATTEN`

## Discovering Data & System Analysis

```sql
-- Browse tables/views
SELECT TABLE_SCHEMA, TABLE_NAME, TABLE_TYPE FROM INFORMATION_SCHEMA."TABLES"
WHERE TABLE_TYPE != 'SYSTEM_TABLE' ORDER BY TABLE_SCHEMA, TABLE_NAME

-- Recent jobs
SELECT * FROM sys.project.jobs_recent ORDER BY submitted_ts DESC LIMIT 20

-- Engine status / Org users
SELECT * FROM sys.project.engines
SELECT * FROM sys.organization.users
```

## Documentation

Dremio Cloud docs: https://docs.dremio.com/dremio-cloud/

Look up syntax with WebFetch when unsure:

| Topic | URL |
|:------|:----|
| SQL Functions | https://docs.dremio.com/dremio-cloud/sql/sql-functions/functions/ |
| SQL Reference | https://docs.dremio.com/dremio-cloud/sql/ |
| COPY INTO | https://docs.dremio.com/dremio-cloud/sql/commands/copy-into-table/ |
| CREATE TABLE | https://docs.dremio.com/dremio-cloud/sql/commands/tables/create-table/ |
| Data Types | https://docs.dremio.com/dremio-cloud/sql/data-types/ |
| REST API | https://docs.dremio.com/dremio-cloud/api/ |

## Best Practices

- If ambiguous which table to use, show what's available and ask.
- On query errors, check: unquoted reserved words, wrong table path, column typos.
- Aggregate first, then drill down. Don't dump hundreds of raw rows.
- Verify column types match before joins.
- Suggest follow-up queries based on results.
