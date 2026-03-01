---
name: dremio-dbt
description: Working with dbt-dremio for data transformations on Dremio Cloud. Use when the user asks about dbt, data modeling, dbt profiles, dbt run, or data transformations with dbt.
user-invocable: false
---

# dbt-dremio

## Profile

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

## Database/schema mapping

dbt-dremio maps `dremio_space` → database, `dremio_space_folder` → schema. The sentinel `no_schema` omits the schema component. **There is no equivalent sentinel for the database position** — `no_schema`, `no_database`, and `""` all emit as literal strings. The namespace must always go in `dremio_space` (database).

In `sources.yml`, use `schema: no_schema` to avoid doubling. If `dremio_space: Medicaid`, do NOT set `schema: Medicaid` — that produces `"Medicaid"."Medicaid"."table"`.

## Running

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
