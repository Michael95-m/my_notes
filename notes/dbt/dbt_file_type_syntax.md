# dbt File Types — Example Syntax Cheat Sheet

*One example per file type used in the dbt-learn course, pulled from the actual lesson content.*

## 1. `profiles.yml` — connection config

```yaml
dbt_project:
  target: dev
  outputs:
    dev:
      type: duckdb
      path: ../database/dev.duckdb
      threads: 4
    prod:
      type: duckdb
      path: ../database/prod.duckdb
      threads: 4
```

## 2. `dbt_project.yml` — project config

```yaml
name: 'dbt_project'
version: '1.0.0'
profile: 'dbt_project'

model-paths:    ["models"]
analysis-paths: ["analyses"]
test-paths:     ["tests"]
seed-paths:     ["seeds"]
macro-paths:    ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

models:
  dbt_project:
    staging:
      +materialized: view
    marts:
      +materialized: table
```

## 3. Model — plain SQL file (`models/staging/stg__customers.sql`)

```sql
select
    id          as customer_id,
    first_name,
    last_name
from {{ source('jaffle_shop', 'raw_customers') }}
```

Mart model using `ref()` + inherited folder materialization (`models/marts/dim_customers.sql`):

```sql
select
    customer_id,
    first_name,
    last_name
from {{ ref('stg__customers') }}
```

Model with an explicit inline config override (`{{ config() }}` at the top):

```sql
{{ config(materialized='table') }}

select
    order_id,
    sum(amount)              as total_amount,
    count(*)                 as payment_count
from {{ ref('stg__payments') }}
group by order_id
```

## 4. `_sources.yml` — declaring raw tables

```yaml
version: 2

sources:
  - name: jaffle_shop
    description: "Application data from the Jaffle Shop app database"
    schema: main
    tables:
      - name: raw_customers
      - name: raw_orders

  - name: stripe
    description: "Payment data landed from the Stripe payment processor"
    schema: main
    tables:
      - name: raw_payments
```

## 5. `_models.yml` — schema YAML (descriptions + tests)

```yaml
version: 2

models:
  - name: stg__customers
    description: "One row per customer from the Jaffle Shop application database."
    columns:
      - name: customer_id
        description: "Surrogate primary key from raw_customers.id."
        tests:
          - unique
          - not_null
      - name: first_name
      - name: last_name

  - name: fct_orders
    description: "One row per order, denormalized with customer attributes and payment totals."
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id
      - name: status
        tests:
          - accepted_values:
              values: [placed, shipped, completed, return_pending, returned]
              config:
                severity: warn
                where: "order_date >= '2018-01-01'"
```

## 6. Seed — CSV file (`seeds/order_status_descriptions.csv`)

```csv
status,description
placed,Order has been placed but not yet shipped
shipped,Order has shipped to the customer
completed,Order delivered and confirmed received
return_pending,Customer has indicated a return; awaiting receipt
returned,Item has been returned to the warehouse
```

## 7. Singular test — `tests/assert_no_negative_payment_totals.sql`

```sql
-- A singular test passes when this query returns 0 rows.
select order_id, total_amount
from {{ ref('int_order_payments') }}
where total_amount < 0
```

## 8. Custom generic test — `tests/generic/non_negative.sql`

```sql
{% test non_negative(model, column_name) %}

select *
from {{ model }}
where {{ column_name }} < 0

{% endtest %}
```

Applying it in schema YAML (no package prefix — it's project-local):

```yaml
columns:
  - name: total_amount
    tests:
      - non_negative
```

## 9. Analysis — `analyses/top_customers_by_revenue.sql` (compiles, never materializes)

```sql
-- Top 10 customers by lifetime order revenue.
select
    c.customer_id,
    c.first_name,
    c.last_name,
    count(o.order_id) as order_count,
    sum(o.amount)     as lifetime_revenue
from {{ ref('fct_orders') }} o
join {{ ref('dim_customers') }} c using (customer_id)
group by 1, 2, 3
order by lifetime_revenue desc
limit 10
```

## 10. Snapshot — `snapshots/orders_snapshot.sql` (SCD Type 2)

```sql
{% snapshot orders_snapshot %}

{{
    config(
        target_schema='snapshots',
        unique_key='order_id',
        strategy='check',
        check_cols=['status']
    )
}}

select * from {{ ref('stg__orders') }}

{% endsnapshot %}
```

## 11. Macro — `macros/cents_to_dollars.sql`

```sql
{% macro cents_to_dollars(column_name, decimal_places=2) -%}
round( 1.0 * {{ column_name }} / 100, {{ decimal_places }} )
{%- endmacro %}
```

Calling it from a model:

```sql
select
    id                                  as payment_id,
    order_id,
    payment_method,
    {{ cents_to_dollars('amount') }}    as amount
from {{ source('stripe', 'raw_payments') }}
```

## 12. Incremental model — `models/marts/fct_order_events.sql`

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

select
    order_id,
    customer_id,
    order_date,
    status
from {{ ref('stg__orders') }}

{% if is_incremental() %}
where order_date > (select max(order_date) from {{ this }})
{% endif %}
```

## 13. Exposures — `models/_exposures.yml`

```yaml
version: 2

exposures:
  - name: orders_dashboard
    label: "Orders Operational Dashboard"
    type: dashboard
    maturity: high
    url: https://example.com/dashboards/orders
    description: "Daily order volume, revenue, and status breakdown."
    depends_on:
      - ref('fct_orders')
      - ref('dim_customers')
    owner:
      name: Data Team
      email: data@example.com
```

## 14. `packages.yml` — installing community packages

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]
  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<0.11.0"]
```

Calling a package macro/test from a model or schema YAML:

```sql
select
    ...,
    {{ dbt_utils.generate_surrogate_key(['o.order_id', 'o.customer_id']) }} as order_key
from {{ ref('stg__orders') }} o
```

```yaml
columns:
  - name: amount
    tests:
      - dbt_expectations.expect_column_values_to_be_between:
          min_value: 0
          max_value: 100
```

## 15. Mart promoted from an analysis — import → logic → final CTE pattern

```sql
{{ config(materialized='table') }}

with

customers as (
    select * from {{ ref('dim_customers') }}
),

orders as (
    select * from {{ ref('fct_orders') }}
),

customer_revenue as (
    select
        c.customer_id,
        c.first_name,
        c.last_name,
        sum(o.amount) as lifetime_revenue,
        count(o.order_id) as lifetime_orders
    from customers c
    left join orders o on c.customer_id = o.customer_id
    group by c.customer_id, c.first_name, c.last_name
),

final as (
    select * from customer_revenue
    order by lifetime_revenue desc
)

select * from final
```

Model-level `meta:`/`tags:` metadata (schema YAML):

```yaml
- name: mart_customer_revenue
  description: "Customer-grain lifetime revenue and order count."
  config:
    meta:
      owner: data_eng
      pii: false
    tags:
      - revenue
      - finance
```
