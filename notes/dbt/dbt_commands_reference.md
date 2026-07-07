# dbt Commands — What, When, and Shortcomings

*Every dbt CLI command referenced across the dbt-learn course.*

## `dbt --version`

- **What**: Prints the installed `dbt-core` version and any adapter plugins (e.g. `duckdb`) under `Plugins`.

- **When**: Right after installation, to confirm the CLI and adapter are both present.

- **Shortcoming**: Doesn't test the actual warehouse connection or `profiles.yml`. A clean version check tells you nothing about whether `dbt debug` will pass.

## `dbt init <project_name>`

- **What**: Scaffolds a new project — `dbt_project.yml`, `models/`, `seeds/`, `snapshots/`, `macros/`, `tests/`, `analyses/`. Run once per project, ever.

- **When**: At the very start of a new project.

- **Shortcoming**: Ships a sample `models/example/` folder that has to be deleted manually (one of its models emits a null row that can break later tests). Run from inside the project directory and you get a nested `dbt_project/dbt_project/`. It may also write a stray `~/.dbt/profiles.yml` you don't actually want.

## `dbt debug`

- **What**: Checks — in order — that dbt can find `dbt_project.yml`, parse `profiles.yml`, match the project to a profile, open a connection, and run `select 1`. Prints `All checks passed!` or the exact step that broke.

- **When**: The first thing to run whenever something "isn't working," before touching model code.

- **Shortcoming**: Only a connectivity smoke test — passing `dbt debug` doesn't guarantee models will compile, or that your user has the warehouse permissions a real build needs.

## `dbt deps`

- **What**: Reads `packages.yml` and clones the listed packages into `dbt_packages/`.

- **When**: Immediately after adding or changing an entry in `packages.yml`, and in every fresh CI environment (packages aren't committed to git).

- **Shortcoming**: Editing `packages.yml` installs nothing by itself — forgetting the follow-up `dbt deps` is one of the most common "why is this macro undefined" mistakes.

## `dbt parse`

- **What**: Parses every model, test, macro, snapshot, and exposure file and (re)builds `target/manifest.json`. Does **not** touch the warehouse.

- **When**: The fastest sanity check for a YAML/Jinja syntax error before running a full build.

- **Shortcoming**: Purely structural — it won't catch a bad column name or a SQL type mismatch, since it never actually executes anything against the warehouse.

## `dbt compile`

- **What**: Renders Jinja (`ref()`, `source()`, macros, `config()`) into raw SQL and writes it to `target/compiled/`. Does not execute it.

- **When**: To see exactly what a model or analysis will send to the warehouse, or to produce a paste-ready `.sql` file from an analysis.

- **Shortcoming**: Rendering successfully doesn't mean the SQL will run — a compiled query can still fail at execution time (e.g. referencing a table/column that doesn't actually exist yet).

## `dbt run`

- **What**: Executes models — issues `create or replace view/table` (or the incremental `insert`/`merge`) against the warehouse.

- **When**: Iterating on one model's SQL during development (`dbt run --select <model>`), or a quick full rebuild.

- **Shortcoming**: Skips seeds, snapshots, and tests entirely. A green `dbt run` says nothing about data quality, and it doesn't gate downstream models on test failures the way `dbt build` does.

## `dbt test`

- **What**: Runs generic (YAML) and singular (`tests/*.sql`) tests. A test passes when its underlying query returns 0 rows.

- **When**: After models exist, to check assumptions — uniqueness, foreign keys, accepted values, custom business rules.

- **Shortcoming**: Needs the models it tests to already be built; running it standalone against stale or missing tables just errors out. It also doesn't rebuild anything itself.

## `dbt seed`

- **What**: Loads CSVs under `seeds/` into the warehouse as tables, inferring column types from the data.

- **When**: For small, slow-changing reference/lookup data that's versioned in git (status codes, country mappings).

- **Shortcoming**: Type inference only samples roughly the first ~1000 rows, so a messy CSV can get the wrong types. It's also commonly misused as a general data-loading tool — it's the wrong fit for large or frequently-changing "raw" data (that belongs behind a real ingestion tool + `source()`).

## `dbt snapshot`

- **What**: Runs the SCD Type 2 logic defined in `snapshots/*.sql`, appending a new row (with `dbt_valid_from`/`dbt_valid_to`) whenever a tracked column changes.

- **When**: Any time you need to preserve history for a table that gets overwritten in place (e.g. order status).

- **Shortcoming**: Only detects a change at the moment you run it — if a value changes twice between two snapshot runs, the intermediate state is lost forever. It also has to be scheduled deliberately; nothing runs it automatically except being included in `dbt build`.

## `dbt build`

- **What**: Runs seeds, models, snapshots, and tests together **in DAG order**, in one command — and blocks a node's downstream dependents if its tests fail.

- **When**: The default "is the whole project healthy" command — CI checks, production deploys, and any time tests exist in the project.

- **Shortcoming**: On a large project, a full `dbt build` re-verifies everything, which gets slow — needs scoping (`--select <model>+`) or state selection (`--select state:modified+`) to stay fast in CI.

## `dbt docs generate`

- **What**: Refreshes `manifest.json` and generates `catalog.json` (column-level metadata pulled by querying the warehouse directly).

- **When**: After a build, before serving or publishing the docs site.

- **Shortcoming**: Needs live warehouse access to introspect tables for `catalog.json` — if the connection fails, it can silently skip catalog generation with just a warning instead of a hard error.

## `dbt docs serve`

- **What**: Starts a local static-site server rendering the lineage graph, column docs, and compiled SQL from `manifest.json` + `catalog.json`.

- **When**: Browsing lineage, onboarding a new analyst, or checking impact before changing/dropping a column.

- **Shortcoming**: Only shows what dbt itself knows (schema, SQL, declared descriptions) — it doesn't capture the business "why" behind logic unless someone wrote it into a `description:`, and it assumes a dbt-literate audience.

## `dbt show`

- **What**: Runs a model's (or an `--inline` query's) compiled SQL as a **read-only** query and prints the first N rows. No `create`/`insert`/`merge` — nothing changes in the warehouse.

- **When**: Quick previews while iterating on a model, or previewing an `ephemeral` model (the only way to see its data at all, since it has no warehouse object).

- **Shortcoming**: Still executes the *full* query under the hood — `--limit 5` only caps the printed output, not the runtime, so it's not free on a model that aggregates a huge table.

## `dbt ls` (alias `dbt list`)

- **What**: Lists resources (models, sources, tests, exposures, etc.) matching a `--select`/`--resource-type` filter, without running anything.

- **When**: Confirming what landed in the project after a structural change, or sanity-checking a selector before using it in a real `run`/`build`/`test`.

- **Shortcoming**: Purely informational — a node showing up in `dbt ls` doesn't mean it will actually compile or execute cleanly.

## `dbt clean`

- **What**: Deletes the directories listed under `clean-targets:` in `dbt_project.yml` — by default `target/` and `dbt_packages/`.

- **When**: Forcing a genuinely fresh state, or clearing a stale parser cache (`partial_parse.msgpack`) that's causing weird behavior.

- **Shortcoming**: Wipes `dbt_packages/` along with `target/` — the very next command that needs a package macro will fail until you remember to re-run `dbt deps`.

## `dbt retry`

- **What**: Reads `target/run_results.json` from the most recent command, finds the nodes that failed, and re-runs only those (plus downstream), skipping everything that already succeeded.

- **When**: After a large build fails partway through — saves re-running the work that already passed.

- **Shortcoming**: Depends entirely on `run_results.json` reflecting the exact last invocation. If you ran a different command in between (or the file was overwritten), `retry` has nothing accurate to resume from.

## `dbt run-operation <macro_name>`

- **What**: Executes a named macro standalone, completely outside the context of building a model — e.g. issuing `grant select` statements or provisioning an external table.

- **When**: One-off DDL/admin tasks that want dbt's Jinja/macro machinery but aren't themselves a model.

- **Shortcoming**: It just runs arbitrary macro SQL for real, with no built-in dry-run — a wrong argument (bad schema, bad role) executes immediately against the warehouse.

## `dbt source freshness`

- **What**: For any source with a `loaded_at_field` and `freshness:` thresholds configured, runs `select max(loaded_at_field)` and compares it against `warn_after`/`error_after`.

- **When**: First step in a CI pipeline, to fail fast when upstream data is stale before wasting time building on top of it.

- **Shortcoming**: Only works for sources that actually declare `loaded_at_field` and thresholds — most quick/naive source declarations (like this course's) don't have it, so freshness silently does nothing for them.

## `dbt clone`

- **What**: Uses a warehouse's native table-cloning feature (Snowflake, BigQuery, Databricks zero-copy clone) to stand up a fast copy of a schema/environment.

- **When**: Spinning up an ephemeral dev/CI environment quickly by cloning prod tables instead of rebuilding everything from scratch.

- **Shortcoming**: Adapter-specific and not available everywhere — DuckDB (used throughout this course) has no clone support, so it's a "know it exists" command rather than one you can practice hands-on here.

---

## Key modifier flags (not standalone commands, but change behavior significantly)

| Flag | What it does | Shortcoming |
|---|---|---|
| `--select <spec>` | Scopes a command to specific nodes (`--select fct_orders`, `--select +fct_orders` for "this + upstream", `--select staging` for a folder) | Selector syntax is easy to get backwards — `--exclude` inverts the whole set, which surprises people expecting it to just narrow a `--select` |
| `--full-refresh` | Forces incremental models to drop and rebuild from scratch | Expensive if run by accident on a large incremental table — it throws away the whole "only load new rows" benefit for that run |
| `--target <name>` | Switches which `profiles.yml` output (connection) to use, e.g. `--target prod` | Doesn't share state with the default target — everything rebuilds fresh unless you also pass `--state` |
| `--state <path>` (with `state:modified+`) | Diffs the current manifest against a prior one, so only changed models + downstream get selected | Only useful if you remember to snapshot `target/` to a stable path *before* making changes — comparing against the current `target/` itself is a no-op |
| `--inline "<sql>"` (with `dbt show`) | Compiles and runs an ad-hoc Jinja-SQL string without writing a file | Easy to forget it still needs full `ref()`/`source()` Jinja syntax, not raw table names |
