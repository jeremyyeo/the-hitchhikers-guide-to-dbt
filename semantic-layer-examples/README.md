---
---

## dbt Semantic Layer examples

Some examples using https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl

All examples below will use a basic `dbt_project.yml` file setup:

```yml
# dbt_project.yml
name: my_dbt_project
profile: all
version: "1.0.0"

models:
  my_dbt_project:
    +materialized: table
```

### Examples

```sql
-- models/coffee_beans.sql
select 1 as id, 'BR' as region, 10 as weight, '1970-01-01'::date as updated_at
union
select 2 as id, 'BR' as region, 20 as weight, '1970-01-01'::date as updated_at
union
select 3 as id, 'CO' as region, 10 as weight, '1970-01-01'::date as updated_at
union
select 4 as id, 'CO' as region, 50 as weight, '1970-01-01'::date as updated_at
```

```yaml
# models/semantic.yml
models:
  - name: coffee_beans
    time_spine:
      standard_granularity_column: updated_at
    columns:
      - name: updated_at
        granularity: day

semantic_models:
  - name: coffee_beans
    model: ref('coffee_beans')
    defaults:
      agg_time_dimension: updated_at
    entities:
      - name: coffee_bean
        type: primary
        expr: id
    dimensions:
      - name: updated_at
        type: time
        type_params:
          time_granularity: day
      - name: region
        type: categorical
    measures:
      - name: weight_total
        agg: sum
        expr: weight
        create_metric: true
      - name: bean_count
        agg: sum
        expr: 1
        create_metric: true
```

Total weight of beans by region:

```sh
$ dbt sl query --metrics weight_total --group-by coffee_bean__region

| COFFEE_BEAN__REGION | WEIGHT_TOTAL |
--------------------------------------
|                  BR |           30 |
|                  CO |           60 |
--------------------------------------
```

Number of beans with weight of equal to or more than 20:

```sh
$ dbt sl query --metrics bean_count --where "{{ Metric('weight_total', group_by=['coffee_bean']) }} >= 20"

| BEAN_COUNT |
--------------
|          2 |
--------------
```

#### Filters on metrics

```yaml
# models/semantic.yml
models:
  - name: coffee_beans
    time_spine:
      standard_granularity_column: updated_at
    columns:
      - name: updated_at
        granularity: day

semantic_models:
  - name: coffee_beans
    model: ref('coffee_beans')
    defaults:
      agg_time_dimension: updated_at
    entities:
      - name: coffee_bean
        type: primary
        expr: id
    dimensions:
      - name: updated_at
        type: time
        type_params:
          time_granularity: day
      - name: region
        type: categorical
    measures:
      - name: weight
        agg: sum
        expr: weight
      - name: bean_count
        agg: sum
        expr: 1

metrics:
  - name: weight_total_for_co
    label: weight_total_for_co
    type: simple
    type_params:
      measure: weight
    filter: |
      {{ Dimension('coffee_bean__region') }} = 'CO'
  - name: weight_total_for_br
    label: weight_total_for_br
    type: simple
    type_params:
      measure: weight
    filter: |
      {{ Dimension('coffee_bean__region') }} = 'BR'
```

```sh
$ mf query --metrics weight_total_for_co
✔ Success 🦄 - query completed after 1.37 seconds
  WEIGHT_TOTAL_FOR_CO
---------------------
                   60

$ mf query --metrics weight_total_for_br
✔ Success 🦄 - query completed after 0.71 seconds
  WEIGHT_TOTAL_FOR_BR
---------------------
                   30
```

#### Custom dimension name

```yaml
# models/semantic.yml
models:
  - name: coffee_beans
    time_spine:
      standard_granularity_column: updated_at
    columns:
      - name: updated_at
        granularity: day

semantic_models:
  - name: coffee_beans
    model: ref('coffee_beans')
    defaults:
      agg_time_dimension: updated_at
    entities:
      - name: coffee_bean
        type: primary
        expr: id
    dimensions:
      - name: updated_at
        type: time
        type_params:
          time_granularity: day
      - name: origin # custom dimension name
        type: categorical
        expr: region
    measures:
      - name: weight
        agg: sum
        expr: weight

metrics:
  - name: weight_total
    label: weight_total
    type: simple
    type_params:
      measure: weight
  - name: weight_total_for_co
    label: weight_total_for_co
    type: simple
    type_params:
      measure: weight
    filter: |
      {{ Dimension('coffee_bean__origin') }} = 'CO'
```

```sh
$ mf query --metrics weight_total --group-by coffee_bean__origin
✔ Success 🦄 - query completed after 1.25 seconds
COFFEE_BEAN__ORIGIN      WEIGHT_TOTAL
---------------------  --------------
CO                                 60
BR                                 30

$ mf query --metrics weight_total_for_co
✔ Success 🦄 - query completed after 0.66 seconds
  WEIGHT_TOTAL_FOR_CO
---------------------
                   60
```


### New (2026) Semantic Layer Spec

https://docs.getdbt.com/docs/build/latest-metrics-spec

If you want to test the new spec, you need dbt-core 1.12:

```sh
$ pip install dbt-core==1.12.0b2 dbt-snowflake dbt-metricflow
```

```yaml
models:
  - name: coffee_beans
    semantic_model:
      enabled: true
    time_spine:
      standard_granularity_column: updated_at
    columns:
      - name: updated_at
        granularity: day
        dimension:
          type: time
      - name: id
        entity:
          type: primary
          name: coffee_bean
      - name: region
        dimension:
          type: categorical

    agg_time_dimension: updated_at
    metrics:
      - name: weight_total
        agg: sum
        expr: weight
        type: simple
      - name: bean_count
        agg: sum
        expr: 1
        type: simple

saved_queries:
  - name: weight_total_by_region
    query_params:
      metrics:
        - weight_total
      group_by:
        - "Dimension('coffee_beans__region')"

exposures:
  - name: my_dashie
    type: dashboard
    owner:
      name: jeremy
```

#### Filters on metrics

```yaml
# models/semantic.yml
# generated with `dbt-autofix deprecations --semantic-layer`
models:
  - name: coffee_beans
    time_spine:
      standard_granularity_column: updated_at
    columns:
      - name: updated_at
        granularity: day
        dimension:
          type: time
      - name: id
        entity:
          type: primary
          name: coffee_bean
      - name: region
        dimension:
          type: categorical
    semantic_model:
      enabled: true
    agg_time_dimension: updated_at
    metrics:
      - name: weight_total_for_co
        label: weight_total_for_co
        type: simple
        filter: |
          {{ Dimension('coffee_bean__region') }} = 'CO'
        agg: sum
        expr: weight
      - name: weight_total_for_br
        label: weight_total_for_br
        type: simple
        filter: |
          {{ Dimension('coffee_bean__region') }} = 'BR'
        agg: sum
        expr: weight
      - name: bean_count
        agg: sum
        expr: 1
        type: simple
        hidden: true
```

#### Custom dimension name

```yaml
# models/semantic.yml
# generated with `dbt-autofix deprecations --semantic-layer`
models:
  - name: coffee_beans
    time_spine:
      standard_granularity_column: updated_at
    columns:
      - name: updated_at
        granularity: day
        dimension:
          type: time
      - name: id
        entity:
          type: primary
          name: coffee_bean
      - name: region
        dimension:
          type: categorical
          name: origin # custom dimension name
    semantic_model:
      enabled: true
    agg_time_dimension: updated_at
    metrics:
      - name: weight_total
        label: weight_total
        type: simple
        agg: sum
        expr: weight
      - name: weight_total_for_co
        label: weight_total_for_co
        type: simple
        filter: |
          {{ Dimension('coffee_bean__origin') }} = 'CO'
        agg: sum
        expr: weight
```
