# dbt-learn — Short Notes by Stage

*Order follows `curriculum.yaml`. One condensed note per stage — the must-remember points only, one idea per line.*

## Stage 0 — Foundations

- **ETL vs ELT**
  - ELT (Extract, Load, Transform) is the modern default on cloud warehouses.
  - dbt is the **"T"** — it runs entirely as SQL inside the warehouse; it never extracts or loads data.

- **Modern data team**
  - Data Engineer owns E+L (ingestion/infra).
  - Analytics Engineer owns T (dbt: staging→marts, tests, docs) — the role this course trains.
  - Data Analyst owns BI/dashboards on top of the marts.
  - The AE role was invented by Tristan Handy at RJMetrics (2016).

- **Cloud warehouses** (Snowflake, BigQuery, Redshift, Databricks, DuckDB)
  - Decouple storage from compute.
  - Cheap storage + elastic pay-per-use compute is what made ELT (and dbt) economical.

- **dbt itself**
  - A CLI that's a compiler + runner.
  - You write `SELECT` statements ("models"); dbt handles DDL, dependency order (via `ref()`/`source()`), tests, docs, and lineage (a DAG).
  - dbt has no storage or execution engine of its own.

- **Three flavors**
  - dbt **Core** — free CLI, self-hosted (used in this course).
  - dbt **Cloud** — managed IDE/scheduler/CI, paid tiers.
  - dbt **Fusion** — new Rust engine, ~30x faster parsing, in beta as of 2025.

## Stage 1 — Your First dbt Project

- **Install stack**
  - `dbt-core` (CLI) + `dbt-duckdb` (warehouse adapter) + `duckdb` (engine).
  - Verify with `dbt --version`.

- **`dbt init <name>`**
  - Scaffolds the project once: `dbt_project.yml`, `models/`, `seeds/`, `snapshots/`, `macros/`, `tests/`, `analyses/`.
  - Delete the sample `models/example/` afterward.

- **Two separate config files**
  - `dbt_project.yml` = the project (paths, defaults).
  - `profiles.yml` = the connection (warehouse + credentials).
  - `dbt debug` smoke-tests the connection.

- **Ingestion**
  - Landing raw data is normally a data-engineer job.
  - Done here by a stand-in Python script writing `raw_*` tables into DuckDB.

- **A model**
  - One `.sql` file, one `SELECT`, no DDL — dbt wraps it in `CREATE VIEW`/`TABLE`.
  - `dbt run` = parse + compile + execute.
  - `dbt compile` = parse + compile only (no warehouse changes).

- **Sources**
  - `{{ source('name','table') }}` declares externally-landed raw tables in YAML so they become real DAG nodes.
  - Without it, a hardcoded table name is invisible to dbt.
  - `ref()` is the equivalent macro for dbt-built models.

- **Materialization defaults**
  - Set per-folder in `dbt_project.yml` via `+materialized: view/table`.
  - The `+` prefix is required, or dbt silently ignores the config.

## Stage 2 — Sources, Refs & Project Structure

- **Splitting sources**
  - Split one generic source into multiple sources, one per real upstream system (e.g. `jaffle_shop` app DB vs `stripe`).
  - Gives per-system freshness rules, ownership, and clearer lineage.

- **Staging (`stg_`)**
  - One model per source table, same grain as the source.
  - Only renames/casts — no joins, no filters, no business logic.

- **Intermediate (`int_`)**
  - Stepping stones that change grain (aggregations, rollups).
  - Reused by 2+ downstream models; not queried directly by analysts.

- **Marts — Fact (`fct_`) vs Dimension (`dim_`)**
  - Facts = events, one row per event (orders).
  - Dimensions = things, one row per entity (customers).
  - Drive the join from the table whose grain you want; `left join` everything else in and `coalesce()` nulls.

- **`dbt build`**
  - = seed + run + test + snapshot in one DAG-ordered command.
  - A failing test blocks its downstream nodes (unlike separate `dbt run`).

- **`dbt docs generate` + `dbt docs serve`**
  - Render the lineage graph, column docs, and compiled SQL.
  - Built from `manifest.json` + `catalog.json`.

- **Naming convention** (industry standard)
  - `src_ → stg_ → int_ → fct_/dim_`.
  - The prefix tells you layer, role, and expected grain at a glance.

## Stage 3 — Data Quality: Tests, Seeds & Analyses

- **Two test types**
  - Generic — YAML, reusable, attached to a column.
  - Singular — one-off `.sql` file in `tests/`.
  - A test passes when its query returns 0 rows.

- **The 4 built-in generic tests**
  - `unique`, `not_null`, `accepted_values` (enum check), `relationships` (foreign-key integrity via `to: ref(...)` + `field:`).

- **Where tests live**
  - A schema YAML (conventionally `_models.yml`), nested under `columns: - name: x, tests: [...]`.

- **Seeds**
  - Small, rarely-changing reference CSVs committed to git, loaded with `dbt seed`.
  - For lookup tables (country codes, status descriptions) — never for primary/raw data (that's ingestion → sources).

- **Analyses**
  - SQL in `analyses/` that compiles (`dbt compile`) but never materializes.
  - Version-controlled, `ref()`-aware ad-hoc/audit queries that don't touch the warehouse.

## Stage 4 — Materializations, Incremental & Snapshots

- **5 materializations**
  - `view` — default; cheap to build, recomputes every read.
  - `table` — fast reads, costly rebuild.
  - `ephemeral` — inlined as a CTE downstream, never touches the warehouse, can't be tested/queried directly.
  - `materialized_view` — warehouse-managed refresh, not available on DuckDB.
  - `incremental` — loads only new rows after the first run.

- **Config precedence** (highest wins)
  - Inline `{{ config() }}` in the SQL file.
  - YAML config.
  - `dbt_project.yml` folder default.
  - dbt's built-in default.

- **Incremental mechanics**
  - `is_incremental()` is true only when the table already exists, no `--full-refresh` was passed, and `materialized='incremental'`.
  - `{{ this }}` self-references the model — never `ref()` yourself.
  - First run = full build; later runs = filtered insert.

- **Incremental strategies**
  - `append` — blind insert, no dedup.
  - `merge` — upsert via `unique_key` (the default/safe choice).
  - `delete+insert` — merge alternative for warehouses without `MERGE`.
  - `insert_overwrite` — replace whole partitions (BigQuery).
  - `merge` without `unique_key` silently degrades to `append`.

- **Snapshots**
  - SCD Type 2 history for mutable source tables (`{% snapshot %}...{% endsnapshot %}` block).
  - Strategy `check` (watch `check_cols`) or `timestamp` (watch an `updated_at` column).
  - Adds 4 metadata columns: `dbt_scd_id`, `dbt_updated_at`, `dbt_valid_from`, `dbt_valid_to`.
  - The currently-active row has `dbt_valid_to IS NULL`.

## Stage 5 — Jinja, Macros, State & Exposures

- **Jinja's 3 delimiters**
  - `{{ }}` expressions — print a value.
  - `{% %}` statements — control flow (`if`/`for`/`set`/`macro`), no output.
  - `{# #}` comments — stripped entirely.

- **Macros**
  - Reusable Jinja functions in `macros/*.sql`, callable anywhere in the project.
  - The DRY mechanism for repeated SQL snippets (e.g. `cents_to_dollars`).

- **State selection**
  - `dbt build --select state:modified+ --state <path>`.
  - Diffs the current `manifest.json` against a prior one and rebuilds only changed models plus everything downstream.
  - The key CI cost/time optimization.

- **Exposures**
  - YAML nodes (`depends_on: [ref(...)]`, `owner:`) that document downstream consumers (dashboards, notebooks).
  - Show up in lineage — metadata only, no SQL, no materialization.

- **Packages** (e.g. `dbt_utils`)
  - Declared in `packages.yml`, installed via `dbt deps` into `dbt_packages/`.
  - Called with a `<package>.<macro>()` prefix.
  - `generate_surrogate_key` hashes multiple columns into one deterministic, stable key — useful when the natural PK is a column combination.

## Stage 6 — Advanced Testing & the dbt CLI Tour

- **Testing maturity ladder**
  - L1 — no tests.
  - L2 — PK tests only.
  - L3 — ~5 tests/model, FKs + enums covered.
  - L4 — package tests + custom generic tests.
  - L5 — full coverage + `severity`/`store_failures`/freshness tuning.

- **Test `config:` knobs**
  - `severity` — `error` (default) vs `warn` (logs but doesn't block the build).
  - `where` — row-level filter.
  - `error_if`/`warn_if` — threshold-based.
  - `store_failures` — persists bad rows to a queryable table.

- **`dbt_expectations` package**
  - Adds statistical/distribution tests (e.g. `expect_column_values_to_be_between`) beyond the 4 built-ins.
  - Installed like any package via `packages.yml` + `dbt deps`.

- **Custom generic tests**
  - `{% test name(model, column_name) %} ... {% endtest %}` saved under `tests/generic/`.
  - A reusable, project-specific convention applied to any model/column — between a one-off singular test and a universal built-in.

- **`dbt show`**
  - `dbt show --select <model> --limit N` previews a model's compiled query output without materializing it.
  - The only way to preview an ephemeral model.

- **Key CLI commands to know**
  - `parse` — fast syntax/YAML check.
  - `ls` — list DAG nodes.
  - `compile` — render Jinja to a file, no execution.
  - `debug` — test the connection.
  - `clean` — wipes `target/` + `dbt_packages/`.
  - `retry` — resume a failed build from where it stopped.
  - `run-operation` — call a macro standalone, e.g. grants.
  - `source freshness` — checks staleness.

- **`target/` artifacts**
  - `manifest.json` — parsed project + config/deps (what most tooling reads).
  - `run_results.json` — last command's pass/fail/status, overwritten each run.
  - `compiled/` — Jinja-resolved SQL, no DDL.
  - `run/` — the actual SQL sent, DDL wrapper included.

## Stage 7 — Production Patterns & Tooling

- **Deployment patterns**
  - 1-trunk — branch → PR → CI temp schema → merge → prod build; used by ~90% of teams, fast.
  - 2-trunk — adds a persistent `qa` branch/UAT stage for batched releases; higher latency, real staging env.

- **Multi-target `profiles.yml`**
  - Supports multiple named targets (`dev`, `prod`, ...) with different schema/credentials/threads — switch with `--target`.
  - Same code, different destination.
  - Never hardcode schema names — always go through `source()`/`ref()`.

- **State selection**
  - `state:modified+ --state <prod_manifest>` is the main production speed lever.
  - Turns "rebuild everything" into "rebuild the diff."

- **dbt Cloud vs self-hosted**
  - dbt Cloud — managed scheduler/IDE/CI, paid per seat.
  - Self-hosted — GitHub Actions/Airflow + `dbt build --target prod`, free but more setup.
  - Both valid; pick based on ops capacity.

- **Promoting an analysis to a model**
  - Graduate when it's queried repeatedly, a dashboard depends on it, or it needs documentation/contracts.
  - Structure the SQL as import CTEs → logic CTEs → a final CTE (`select * from final`).
  - Add `description:`, `meta:` (owner, pii), and `tags:` (enables `--select tag:x` selection) in the schema YAML, plus tests.

- **Wider tooling landscape** (know they exist)
  - dbt Fusion — Rust engine, faster parse + local SQL validation.
  - dbt Semantic Layer/MetricFlow — define metrics once.
  - dbt MCP — AI-agent integration.
  - Community packages — `dbt_utils`, `dbt_expectations`, `audit_helper`, `dbt_project_evaluator`, `elementary`.
  - dbt Mesh — multi-project splits with `access:`/contracts for large orgs.
