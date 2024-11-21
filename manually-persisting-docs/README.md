---
---

## Manually persisting docs

Databricks example but most adapters work roughly the same way.

When enabling [persist-docs](https://docs.getdbt.com/reference/resource-configs/persist_docs) - there will be some modifications to the DDL statements run during the creation of the relation and after the creation of the relation. Let's have a look:

```sql
-- models/foo.sql
{{ config(materialized='table', persist_docs={'relation': true, 'columns': true}) }}
select 1 id, 'alice' as first_name
```

```yaml
# models/schema.yml
models:
  - name: foo
    description: The foo table.
    columns:
      - name: id
        description: The id.
      - name: first_name
        description: The first name.
```

```sh
$ dbt --debug run -s foo
23:03:50  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        create or replace table `dev`.`dbt_jyeo`.`foo`
      using delta
      comment 'The foo table.'
      as
select 1 id, 'alice' as first_name

23:04:03  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        alter table `dev`.`dbt_jyeo`.`foo` change column
            id
            comment 'The id.';

23:04:05  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        alter table `dev`.`dbt_jyeo`.`foo` change column
            first_name
            comment 'The first name.';
```

What if we want to add those comments in a separate run-operation so we don't do it during model build time. Here's a quick demo on how we can achieve that:

```sql
-- macros/add_comments.sql
{% macro add_comments() %}
    {% if execute %}
        {% set models = graph.nodes.values() | selectattr("resource_type", "equalto", "model") %}
        {% for m in models %}
            /*{# The fqn of the model which we will use later. #}*/
            {% set relation = m.database ~ '.' ~ m.schema ~ '.' ~ m.alias %}
            {% do print('>>> Adding comments to ' ~ relation ~ ' <<<') %}

            /*{# Add model descriptions if there's any. #}*/
            {% set model_description = m.get('description', '') %}
            {% if model_description != '' %}
                {% set add_model_description_ddl -%}
                    comment on table {{ relation }} is '{{ model_description }}';
                {%- endset %}
                {% do run_query(add_model_description_ddl) %}
            {% endif %}

            /*{# Add column descriptions if there's any. #}*/
            {% set model_columns = m.columns.values() %}
            {% for mc in model_columns %}
                {% set column_description = mc.get('description', '') %}
                {% if column_description != '' %}
                    {% set add_column_description_ddl -%}
                        alter table {{ relation }} change column {{ mc.name }} comment '{{ column_description }}';
                    {%- endset %}
                    {% do run_query(add_column_description_ddl) %}
                {% endif %}
            {% endfor %}
            {% do print('=================================================') %}
        {% endfor %}
    {% endif %}
{% endmacro %}
```

```sql
-- models/foo.sql
{{ config(materialized='table') }}
select 1 id, 'alice' as first_name

-- models/bar.sql
{{ config(materialized='table') }}
select 1 id

-- models/baz.sql
{{ config(materialized='table') }}
select 1 id
```

```yaml
# models/schema.yml
models:
  - name: foo
    description: The foo table.
    columns:
      - name: id
        description: The id.
      - name: first_name
        description: The first name.
  - name: bar
    description: The bar table.
  - name: baz
    columns:
      - name: id
        description: The id.
```

Here were not making use of `persist_docs` - therefore when builing the models, those descriptions wont be added:

```sh
$ dbt --debug run
...
02:04:31  3 of 3 START sql table model dbt_jyeo.foo ...................................... [RUN]
...
02:04:31  Using databricks connection "model.my_dbt_project.foo"
02:04:31  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "node_id": "model.my_dbt_project.foo"} */
        create or replace table `dev`.`dbt_jyeo`.`foo`
      using delta
      as
select 1 id, 'alice' as first_name
...
02:04:33  3 of 3 OK created sql table model dbt_jyeo.foo ................................. [OK in 1.99s]
...
```

Then we can make use of our macro in a run-op:

```sh
$ dbt --debug run-operation add_comments
...
>>> Adding comments to dev.dbt_jyeo.foo <<<
02:06:02  On macro_add_comments: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "connection_name": "macro_add_comments"} */

    comment on table dev.dbt_jyeo.foo is 'The foo table.';
  
02:06:06  On macro_add_comments: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "connection_name": "macro_add_comments"} */

    alter table dev.dbt_jyeo.foo change column id comment 'The id.';
  
02:06:07  On macro_add_comments: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "connection_name": "macro_add_comments"} */

    alter table dev.dbt_jyeo.foo change column first_name comment 'The first name.';
  
02:06:08  Databricks adapter: Cursor(session-id=01efa7ad-29ab-1ad0-b114-5c1016366708, command-id=01efa7ad-2c1a-1fe4-a871-1826437d7aa0) - Closing cursor
=================================================

>>> Adding comments to dev.dbt_jyeo.bar <<<

02:06:08  On macro_add_comments: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "connection_name": "macro_add_comments"} */

    comment on table dev.dbt_jyeo.bar is 'The bar table.';
  
02:06:10  Databricks adapter: Cursor(session-id=01efa7ad-29ab-1ad0-b114-5c1016366708, command-id=01efa7ad-2cd1-1942-968a-d40b0805905d) - Closing cursor
=================================================

>>> Adding comments to dev.dbt_jyeo.baz <<<
02:06:10  On macro_add_comments: /* {"app": "dbt", "dbt_version": "1.9.0b4", "dbt_databricks_version": "1.9.0b1", "databricks_sql_connector_version": "3.4.0", "profile_name": "all", "target_name": "db", "connection_name": "macro_add_comments"} */

    alter table dev.dbt_jyeo.baz change column id comment 'The id.';
  
02:06:11  Databricks adapter: Cursor(session-id=01efa7ad-29ab-1ad0-b114-5c1016366708, command-id=01efa7ad-2dc8-1ac6-83c3-3387f6ebe683) - Closing cursor
=================================================
...
```

^ As we can see, we've submitted all the necessary DDL to add the descriptions of our models and columns to the various tables and columns on Databricks. Do a quick check in the Databricks UI:

![alt text](image.png)
