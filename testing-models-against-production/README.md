---
---

## Testing models against production

A Snowflake example of how to test that a model that runs in CI is exactly equal to a model that is in production.

A dbt test will error if there is at least one row of results so we can use the `except` operator - which returns the difference between two tables.

```sql
create or replace table foo as select 1 id;
create or replace table bar as select 1 id;
select * from foo except select * from bar;
-- Query produced no results.

create or replace table bar as select 2 id;
select * from foo except select * from bar;
-- (id: 1)
```

We will be setting up 2 environments / "targets" and will be varying our schemas based on the target. The `database` between the 2 targets will not change - it will simply be the default database set on the environment.
* Production:
    * `target.name == 'prod'`.
    * Database: `db`.
    * Schema: `analytics`.
* CI:
    * `target.name == 'ci'`.
    * Database: `db`.
    * Schema: `dbt_cloud_pr_123_456`.

The dbt project setup:

```sql
-- models/foo.sql
select 1 id

-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {#/* In dbt Cloud CI jobs, target.schema takes on the form of 'dbt_cloud_pr_XXX_YYY' */#}
    {%- if custom_schema_name is none -%}
        {{ target.schema }}
    {%- else -%}
        {%- if target.name == 'ci' -%}
            {{ target.schema }}
        {%- else -%}
            {{ custom_schema_name | trim }}
        {%- endif -%}
    {%- endif -%}
{%- endmacro %}
```

There's a simple model `foo` and the `generate_schema_name` macro is used to resolve the schema where we want our model to go into accordingly. In production runs, the model's schema resolve to `custom_schema_name` (which is given by the `+schema` config on the model). In CI runs, the schema resolves to the `target.schema` value - in dbt Cloud, that value is overridden and takes on the form of `dbt_cloud_pr_XXX_YYY`.

```yaml
# dbt_project.yml
name: my_dbt_project
profile: sf
version: "1.0.0"

models:
  my_dbt_project:
    +materialized: table
    +schema: analytics

# models/schema.yml
models:
  - name: foo
    data_tests:
      - is_equal_to_prod:
          config:
            enabled: "{{ target.name == 'ci' | as_bool }}"
```

What we can see here is that there is a custom generic test `is_equal_to_prod` that we only want to enable in CI runs and not otherwise. The custom generic test:

```sql
-- tests/generic/is_equal_to_prod.sql
{% test is_equal_to_prod(model) %}

{% if execute %}
    {% set production_node = graph.nodes.values() | selectattr("relation_name", "equalto", model | string) | list %}
    {% set production_schema = production_node[0].config.schema or target.schema %}
    {% set production_database = production_node[0].config.database or target.database %}
    {% set production_model = api.Relation.create(database=production_database, schema=production_schema, identifier=model.identifier) %}
    {% do print('DEBUG: Testing ' ~ model ~ ' against ' ~ production_model) %}
{% else %}
    {% set production_model = model %}
{% endif %}

select * from {{ model }}
except 
select * from {{ production_model }}

{% endtest %}
```

First, let's do a production run and look at the logs:

```sh
$ dbt build -t prod
...
00:37:28  Concurrency: 1 threads (target='prod')
00:37:28  
00:37:28  Began running node model.my_dbt_project.foo
00:37:28  1 of 1 START sql table model analytics.foo ..................................... [RUN]
00:37:28  Re-using an available connection from the pool (formerly list_db_analytics, now model.my_dbt_project.foo)
00:37:28  Began compiling node model.my_dbt_project.foo
00:37:28  Writing injected SQL for node "model.my_dbt_project.foo"
00:37:28  Began executing node model.my_dbt_project.foo
00:37:28  Writing runtime sql for node "model.my_dbt_project.foo"
00:37:28  Using snowflake connection "model.my_dbt_project.foo"
00:37:28  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "sf", "target_name": "prod", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table db.analytics.foo
         as
        (select 1 id
        );
00:37:28  Opening a new connection, currently in state closed
00:37:31  SQL status: SUCCESS 1 in 2.734 seconds
00:37:31  On model.my_dbt_project.foo: Close
00:37:32  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '466465a8-e091-4511-a005-abad8b916258', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x165c8a1d0>]}
00:37:32  1 of 1 OK created sql table model analytics.foo ................................ [SUCCESS 1 in 3.36s]
00:37:32  Finished running node model.my_dbt_project.foo
00:37:32  Connection 'master' was properly closed.
00:37:32  Connection 'model.my_dbt_project.foo' was properly closed.
00:37:32  
00:37:32  Finished running 1 table model in 0 hours 0 minutes and 7.99 seconds (7.99s).
00:37:32  Command end result
00:37:32  
00:37:32  Completed successfully
```

All is expected - the model is built where it is meant to and the test didn't execute. Now, let's do a CI run without changing the model code:

```sh
$ dbt build -t ci
...
00:41:01  1 of 2 START sql table model dbt_cloud_pr_123_456.foo .......................... [RUN]
00:41:01  Re-using an available connection from the pool (formerly list_db_dbt_cloud_pr_123_456, now model.my_dbt_project.foo)
00:41:01  Began compiling node model.my_dbt_project.foo
00:41:01  Writing injected SQL for node "model.my_dbt_project.foo"
00:41:01  Began executing node model.my_dbt_project.foo
00:41:01  Writing runtime sql for node "model.my_dbt_project.foo"
00:41:01  Using snowflake connection "model.my_dbt_project.foo"
00:41:01  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "sf", "target_name": "ci", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table db.dbt_cloud_pr_123_456.foo
         as
        (select 1 id
        );
00:41:01  Opening a new connection, currently in state closed
00:41:04  SQL status: SUCCESS 1 in 2.392 seconds
00:41:04  On model.my_dbt_project.foo: Close
00:41:04  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'c57d6f42-8a94-4381-ab3a-d249af8df0ff', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x105db23d0>]}
00:41:04  1 of 2 OK created sql table model dbt_cloud_pr_123_456.foo ..................... [SUCCESS 1 in 3.10s]
00:41:04  Finished running node model.my_dbt_project.foo
00:41:04  Began running node test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01
00:41:04  2 of 2 START test is_equal_to_prod_foo_ ........................................ [RUN]
00:41:04  Re-using an available connection from the pool (formerly model.my_dbt_project.foo, now test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01)
00:41:04  Began compiling node test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01
DEBUG: Testing db.dbt_cloud_pr_123_456.foo against db.analytics.foo
00:41:04  Writing injected SQL for node "test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01"
00:41:04  Began executing node test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01
00:41:04  Writing runtime sql for node "test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01"
00:41:04  Using snowflake connection "test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01"
00:41:04  On test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "sf", "target_name": "ci", "node_id": "test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01"} */
select
      count(*) as failures,
      count(*) != 0 as should_warn,
      count(*) != 0 as should_error
    from (
 
select * from db.dbt_cloud_pr_123_456.foo
except 
select * from db.analytics.foo
 
    ) dbt_internal_test
00:41:04  Opening a new connection, currently in state closed
00:41:06  SQL status: SUCCESS 1 in 2.076 seconds
00:41:06  On test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01: Close
00:41:07  2 of 2 PASS is_equal_to_prod_foo_ .............................................. [PASS in 2.73s]
00:41:07  Finished running node test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01
00:41:07  Connection 'master' was properly closed.
00:41:07  Connection 'test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01' was properly closed.
00:41:07  
00:41:07  Finished running 1 table model, 1 test in 0 hours 0 minutes and 13.43 seconds (13.43s).
00:41:07  Command end result
00:41:07  
00:41:07  Completed successfully
```

Our CI model is built into the CI schema and it is tested against the production model. Because there are no differences, no rows are returned and therefore, the test is passing. Let's quickly change our model code and do another CI run:

```sql
-- models/foo.sql
select 2 id
```

```sh
$ dbt build -t ci
...
00:43:15  1 of 2 START sql table model dbt_cloud_pr_123_456.foo .......................... [RUN]
00:43:15  Re-using an available connection from the pool (formerly list_db_dbt_cloud_pr_123_456, now model.my_dbt_project.foo)
00:43:15  Began compiling node model.my_dbt_project.foo
00:43:15  Writing injected SQL for node "model.my_dbt_project.foo"
00:43:15  Began executing node model.my_dbt_project.foo
00:43:15  Writing runtime sql for node "model.my_dbt_project.foo"
00:43:15  Using snowflake connection "model.my_dbt_project.foo"
00:43:15  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "sf", "target_name": "ci", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table db.dbt_cloud_pr_123_456.foo
         as
        (select 2 id
        );
00:43:15  Opening a new connection, currently in state closed
00:43:17  SQL status: SUCCESS 1 in 2.477 seconds
00:43:17  On model.my_dbt_project.foo: Close
00:43:18  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'fcb1c349-3e85-43ad-9917-3e389f8b2f74', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x121329090>]}
00:43:18  1 of 2 OK created sql table model dbt_cloud_pr_123_456.foo ..................... [SUCCESS 1 in 3.15s]
00:43:18  Finished running node model.my_dbt_project.foo
00:43:18  Began running node test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01
00:43:18  2 of 2 START test is_equal_to_prod_foo_ ........................................ [RUN]
00:43:18  Re-using an available connection from the pool (formerly model.my_dbt_project.foo, now test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01)
00:43:18  Began compiling node test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01
DEBUG: Testing db.dbt_cloud_pr_123_456.foo against db.analytics.foo
00:43:18  Writing injected SQL for node "test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01"
00:43:18  Began executing node test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01
00:43:18  Writing runtime sql for node "test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01"
00:43:18  Using snowflake connection "test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01"
00:43:18  On test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "sf", "target_name": "ci", "node_id": "test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01"} */
select
      count(*) as failures,
      count(*) != 0 as should_warn,
      count(*) != 0 as should_error
    from (

select * from db.dbt_cloud_pr_123_456.foo
except 
select * from db.analytics.foo

    ) dbt_internal_test
00:43:18  Opening a new connection, currently in state closed
00:43:20  SQL status: SUCCESS 1 in 1.696 seconds
00:43:20  On test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01: Close
00:43:20  2 of 2 FAIL 1 is_equal_to_prod_foo_ ............................................ [FAIL 1 in 2.36s]
00:43:20  Finished running node test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01
00:43:20  Connection 'master' was properly closed.
00:43:20  Connection 'test.my_dbt_project.is_equal_to_prod_foo_.9191aeee01' was properly closed.
00:43:20  
00:43:20  Finished running 1 table model, 1 test in 0 hours 0 minutes and 11.01 seconds (11.01s).
00:43:20  Command end result
00:43:20  
00:43:20  Completed with 1 error and 0 warnings:
00:43:20  
00:43:20  Failure in test is_equal_to_prod_foo_ (models/schema.yml)
00:43:20    Got 1 result, configured to fail if != 0
00:43:20  
00:43:20    compiled code at target/compiled/my_dbt_project/models/schema.yml/is_equal_to_prod_foo_.sql
00:43:20  
00:43:20  Done. PASS=1 WARN=0 ERROR=1 SKIP=0 TOTAL=2
```

Now we rebuilt `foo` in our CI schema - but of course, since the table has changed compared to the table that exist in production (`db.analytics.foo`) - the `select ... except ...` query returned some result - which causes the test to emit an error as expected.
