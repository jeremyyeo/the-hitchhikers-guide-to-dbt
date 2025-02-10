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

