---
---

## Custom materialization

Some sample custom materializations - https://docs.getdbt.com/guides/create-new-materializations?step=2

### A model that builds more than 1 table

```sql
-- macros/custom_table.sql
{% materialization custom_table, adapter='snowflake' %}
    /*{# Adapted from https://github.com/dbt-labs/dbt-adapters/blob/main/dbt-snowflake/src/dbt/include/snowflake/macros/materializations/table.sql #}*/

    {%- set tenants = config.get('tenants') -%}
    {%- set language = model['language'] -%}
    {%- set target_relation = api.Relation.create(identifier=model['name'], schema=schema, database=database, type='table') -%}

    {%- for tenant in tenants -%}
        {%- set identifier = model['name'] ~ '_' ~ tenant -%}
        {%- set existing_relation = adapter.get_relation(database=database, schema=schema, identifier=identifier) -%}
        {%- set target_relation = api.Relation.create(identifier=identifier, schema=schema, database=database, type='table') -%}
        {% call statement('main') -%}
           create or replace table {{ target_relation }} as (select 100 as sales);
        {%- endcall %}
    {%- endfor -%}

    {{ return({'relations': [target_relation]}) }}
{% endmaterialization %}

-- models/sales.sql
/* see config in sales.yml file */
```

```yaml
# models/sales.yml
models:
  - name: sales
    config:
      materialized: custom_table
      tenants:
        - aperture_science
        - cyberdyne_systems
        - dunder_mifflin
        - vault_tec
```

```sh
$ dbt --debug build

02:51:58  1 of 1 START sql custom_table model sch.sales .................................. [RUN]
02:51:58  On model.my_dbt_project.sales: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.sales"} */
create or replace table db.sch.sales_aperture_science as (select 100 as sales);
02:51:59  SQL status: SUCCESS 1 in 1.224 seconds
02:51:59  On model.my_dbt_project.sales: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.sales"} */
create or replace table db.sch.sales_cyberdyne_systems as (select 100 as sales);
02:52:01  SQL status: SUCCESS 1 in 1.304 seconds
02:52:01  On model.my_dbt_project.sales: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.sales"} */
create or replace table db.sch.sales_dunder_mifflin as (select 100 as sales);
02:52:02  SQL status: SUCCESS 1 in 1.319 seconds
02:52:02  On model.my_dbt_project.sales: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.sales"} */
create or replace table db.sch.sales_vault_tec as (select 100 as sales);
02:52:03  SQL status: SUCCESS 1 in 1.005 seconds
02:52:03  1 of 1 OK created sql custom_table model sch.sales ............................. [SUCCESS 1 in 4.90s]
```

^ 1 table per `tenant` created.
