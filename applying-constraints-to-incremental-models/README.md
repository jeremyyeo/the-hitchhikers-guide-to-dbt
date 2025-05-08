---
---

## Applying constraints to incremental models on

By default, dbt only applies constraints whenever the table is being created (i.e. for the first time / full-refresh flagged for incremental models). What happens then is if you retroactively edit the schema yaml file to add constraints - what will happen then is dbt will not apply the constraints. Let's see how we can write a macro that will apply those constraints in a model post-hook.

> The example below is on Databricks.

```sql
-- models/foo.sql
{{
    config(
        materialized='incremental',
        on_schema_change='append_new_columns'
    )
}}

select 1 id
```

```yaml
# models/schema.yml
models:
  - name: foo
```

We have got an incremental model like the one above and we're going to do a first build:

```sh
$ dbt build
03:46:39  1 of 1 START sql incremental model dbt_jyeo_prod.foo ........................... [RUN]
03:46:39  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        create or replace table `dev`.`dbt_jyeo_prod`.`foo`
      using delta
      as
select 1 id
03:46:41  SQL status: OK in 2.100 seconds
03:46:41  1 of 1 OK created sql incremental model dbt_jyeo_prod.foo ...................... [OK in 2.19s]
```

Next, we try adding a constraint to it:

```yaml
# models/schema.yml
models:
  - name: foo
    config:
      contract:
        enforced: true
    columns:
      - name: id
        data_type: int
        constraints:
          - type: not_null
          - type: check
            expression: "id > 0"
```

We then build our model again:

```sh
$ dbt build
03:48:22  1 of 1 START sql incremental model dbt_jyeo_prod.foo ........................... [RUN]
03:48:22  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
SELECT tag_name, tag_value
  FROM `system`.`information_schema`.`table_tags`
  WHERE catalog_name = 'dev'
    AND schema_name = 'dbt_jyeo_prod'
    AND table_name = 'foo'
03:48:23  SQL status: OK in 0.620 seconds
03:48:23  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
SHOW TBLPROPERTIES `dev`.`dbt_jyeo_prod`.`foo`
03:48:23  SQL status: OK in 0.530 seconds
03:48:23  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
    create or replace temporary view `foo__dbt_tmp` as
select 1 id
03:48:24  SQL status: OK in 0.390 seconds
03:48:24  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
describe table `foo__dbt_tmp`
03:48:24  SQL status: OK in 0.600 seconds
03:48:24  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
describe table `dev`.`dbt_jyeo_prod`.`foo`
03:48:25  SQL status: OK in 0.460 seconds
03:48:25  Databricks adapter: Cursor(session-id=01f02bbf-47d6-18aa-9954-534e35060b15, command-id=01f02bbf-4b81-18d1-a92a-f8c6991d3d1d) - Closing
03:48:25
    In `dev`.`dbt_jyeo_prod`.`foo`:
        Schema changed: False
        Source columns not in target: []
        Target columns not in source: []
        New column types: []
03:48:25  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
describe table `dev`.`dbt_jyeo_prod`.`foo`
03:48:25  SQL status: OK in 0.470 seconds
03:48:25  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
describe table `foo__dbt_tmp`
03:48:26  SQL status: OK in 0.520 seconds
03:48:26  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
-- back compat for old kwarg name
    merge
    into
        `dev`.`dbt_jyeo_prod`.`foo` as DBT_INTERNAL_DEST
    using
        `foo__dbt_tmp` as DBT_INTERNAL_SOURCE
    on
        FALSE
    when matched
        then update set
            `id` = DBT_INTERNAL_SOURCE.`id`
    when not matched
        then insert
            (`id`) VALUES (DBT_INTERNAL_SOURCE.`id`)
03:48:29  SQL status: OK in 2.910 seconds
03:48:29  1 of 1 OK created sql incremental model dbt_jyeo_prod.foo ...................... [OK in 6.70s]
```

As we can see - there was not any SQL statements that tried to add constraints to the foo model. Let's make dbt build the model from scratch again:

```sh
$ dbt build --full-refresh

03:51:24  1 of 1 START sql incremental model dbt_jyeo_prod.foo ........................... [RUN]
03:51:24  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
select * from (
select 1 id
    ) as __dbt_sbq
    where false
    limit 0
03:51:25  SQL status: OK in 0.340 seconds
03:51:25  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
select * from (
        select

    cast(null as int)
 as id
    ) as __dbt_sbq
    where false
    limit 0
03:51:25  SQL status: OK in 0.310 seconds
03:51:25  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        create or replace table `dev`.`dbt_jyeo_prod`.`foo`
      using delta
      as
    select id
    from (
select 1 id
    ) as model_subq
03:51:27  SQL status: OK in 1.930 seconds
03:51:27  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        alter table `dev`.`dbt_jyeo_prod`.`foo` change column id set not null ;
03:51:28  SQL status: OK in 1.320 seconds
03:51:28  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        alter table `dev`.`dbt_jyeo_prod`.`foo` add constraint 1adc2df6b709a9f57a9f77192d42642d check (id > 0);
03:51:30  SQL status: OK in 1.700 seconds
03:51:30  1 of 1 OK created sql incremental model dbt_jyeo_prod.foo ...................... [OK in 5.81s]
```

As we can see, this is consistent with what was mentioned - dbt will only emit constraint DDL statements if the model is being created for the first time. This can be costly if the incremental table is very large. Let's try to make dbt add constraints even if it's not the first time building the model with a macro that we can use in our models post hook.

```sql
-- macros/apply_constraints.sql
{% macro apply_constraints() -%}
  {% if execute %}
    {% if model.config.contract.enforced %}
      {% for col in model.columns %}
        {% if model.columns[col].constraints | length > 0 %}
          {% set all_constraints = model.columns[col].constraints %}
          {% for constraint in all_constraints %}
            {% if constraint.type == 'not_null' %}
              {% set ddl -%}
                alter table {{ this }} alter column {{ col }} set not null;
              {% endset %}
              {% do run_query(ddl) %}
            {% elif constraint.type == 'check' %}
              {% set ddl -%}
                alter table {{ this }} add constraint const_{{ run_started_at.strftime('%s') }} check ({{ constraint.expression }});
              {% endset %}
              {% do run_query(ddl) %}
             {% endif %}
          {% endfor %}
        {% endif %}
      {% endfor %}
    {% endif %}
  {% endif %}
{%- endmacro %}

```

```sql
-- models/foo.sql
{{
    config(
        materialized='incremental',
        on_schema_change='append_new_columns',
        post_hook='{{ apply_constraints() }}'
    )
}}

select 1 id
```

```yaml
# models/schema.yml
models:
  - name: foo
```

Again, we do an initial build without specifying any constraints:

```sh
$ dbt build

03:56:15  1 of 1 START sql incremental model dbt_jyeo_prod.foo ........................... [RUN]
03:56:15  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        create or replace table `dev`.`dbt_jyeo_prod`.`foo`
      using delta
      as
select 1 id
03:56:17  SQL status: OK in 2.420 seconds
03:56:17  1 of 1 OK created sql incremental model dbt_jyeo_prod.foo ...................... [OK in 2.51s]
```

^ All the same stuff as before. Let's add those contraints back in:

```yaml
# models/schema.yml
models:
  - name: foo
    config:
      contract:
        enforced: true
    columns:
      - name: id
        data_type: int
        constraints:
          - type: not_null
          - type: check
            expression: "id > 0"
```

And do that incremental build again:

```sh
$ dbt build
03:57:51  1 of 1 START sql incremental model dbt_jyeo_prod.foo ........................... [RUN]
<TRUNCATED>
03:57:55  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
-- back compat for old kwarg name
    merge
    into
        `dev`.`dbt_jyeo_prod`.`foo` as DBT_INTERNAL_DEST
    using
        `foo__dbt_tmp` as DBT_INTERNAL_SOURCE
    on
        FALSE
    when matched
        then update set
            `id` = DBT_INTERNAL_SOURCE.`id`
    when not matched
        then insert
            (`id`) VALUES (DBT_INTERNAL_SOURCE.`id`)
03:57:57  SQL status: OK in 2.500 seconds
03:57:57  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
    alter table `dev`.`dbt_jyeo_prod`.`foo` alter column id set not null;
03:57:59  SQL status: OK in 1.320 seconds
03:57:59  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.4", "dbt_databricks_version": "1.9.7", "databricks_sql_connector_version": "3.7.1", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
    alter table `dev`.`dbt_jyeo_prod`.`foo` add constraint const_1746633465 check (id > 0);
03:58:00  SQL status: OK in 1.760 seconds
03:58:00  1 of 1 OK created sql incremental model dbt_jyeo_prod.foo ...................... [OK in 9.38s]
```

^ The key difference now is that we're using our post hooks to add those contraints after the model has been merged into. This means we can add contraints without having to do a full refresh - unlike with the out of the box contraints config.

Note that this only serves as an example on how to add contraints without having to do a full refresh. It is not a 100% complete solution - for example, this will always apply contraints everytime the model is ran - regardless of if those contraints already exist or not, it also doesn't do anything with the `data_type` config. An enterprising analytics engineer should extend this example to perhaps check if those constraints already exist before blanket applying them (like what we're doing in this example).
