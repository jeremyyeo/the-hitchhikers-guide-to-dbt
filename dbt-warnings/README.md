---
---

## dbt warnings

https://docs.getdbt.com/reference/global-configs/warnings

A catalog of warnings and what they mean with examples.

### Notes

1. These warnings show up at the start of the logs even if your selection does not include the node that dbt is warning you about. For example, you may have done `dbt seed` or `dbt run --select foo` but dbt will still emit all the warnings even though the warnings were not relevant to seeds or model `foo`.
2. Due to partial parsing, warnings may show up in one run but not in a subsequent one even if the criteria for the warning has not yet been resolved - https://github.com/dbt-labs/dbt-core/issues/10323

### Configuration paths exist in your dbt_project.yml file which do not apply to any resources.

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
config-version: 2
version: "1.0.0"

models:
  my_dbt_project:
    +materialized: table
    marts:
      +materialized: view
```

```sql
-- models/foo.sql
select 1 id
```

```sh
$ dbt run
00:49:55  Running with dbt=1.6.14
00:49:55  Registered adapter: postgres=1.6.14
00:49:55  Unable to do partial parsing because saved manifest not found. Starting full parse.
00:49:55  [WARNING]: Configuration paths exist in your dbt_project.yml file which do not apply to any resources.
There are 1 unused configuration paths:
- models.my_dbt_project.marts
00:49:55  Found 1 model, 0 sources, 0 exposures, 0 metrics, 352 macros, 0 groups, 0 semantic models
00:49:55
00:49:55  Concurrency: 4 threads (target='pg')
00:49:55
00:49:55  1 of 1 START sql table model public.foo ........................................ [RUN]
00:49:55  1 of 1 OK created sql table model public.foo ................................... [SELECT 1 in 0.07s]
00:49:55
00:49:55  Finished running 1 table model in 0 hours 0 minutes and 0.20 seconds (0.20s).
00:49:55
00:49:55  Completed successfully
00:49:55
00:49:55  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

^ Here I'm telling dbt to apply `materialized: view` to models in a folder `models/marts/` which does not exist and dbt is warning me of exactly that. The folder can simply not exist literally or because of incorrect casing - i.e. the actual folder name is `Marts` but you tried to apply the config on folder `marts`.

### Did not find matching node for patch with name '...' in the '...' section of file '....yml'

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
config-version: 2
version: "1.0.0"

models:
   my_dbt_project:
      +materialized: table

# models/schema.yml
version: 2
models:
  - name: foo
    description: Lorem ipsum
  - name: bar
    description: Lorem ipsum
```

```sql
-- models/foo.sql
select 1 id
```

```sh
$ dbt run
01:21:31  Running with dbt=1.6.14
01:21:31  Registered adapter: postgres=1.6.14
01:21:31  Unable to do partial parsing because saved manifest not found. Starting full parse.
01:21:31  [WARNING]: Did not find matching node for patch with name 'bar' in the 'models' section of file 'models/schema.yml'
01:21:31  Found 1 model, 0 sources, 0 exposures, 0 metrics, 352 macros, 0 groups, 0 semantic models
01:21:31
01:21:31  Concurrency: 4 threads (target='pg')
01:21:31
01:21:31  1 of 1 START sql table model public.foo ........................................ [RUN]
01:21:31  1 of 1 OK created sql table model public.foo ................................... [SELECT 1 in 0.08s]
01:21:31
01:21:31  Finished running 1 table model in 0 hours 0 minutes and 0.18 seconds (0.18s).
01:21:31
01:21:31  Completed successfully
01:21:31
01:21:31  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

We told dbt that there exist a model `bar` in the `schema.yml` file but in reality - it does not exist. Note it can also not exist due to a mismatched case - i.e. in the `schema.yml` file we did `name: Bar` (uppercase "B") but the model is actually `bar.sql` (lowercase "b").

### Explicit transactional logic should be used only to wrap DML logic (MERGE, DELETE, UPDATE, etc)

The full warning:

```
Snowflake adapter: [WARNING]: Explicit transactional logic should be used only to wrap DML
logic (MERGE, DELETE, UPDATE, etc). The keywords BEGIN; and COMMIT; should be
placed directly before and after your DML statement, rather than in separate
statement calls or run_query() macros.
```

Which is usually accompanied by an error:

```
cannot access local variable 'connection' where it is not associated with a value
```

Typically on Snowflake and happens when you accidentally return a SQL comment to a hook (pre/post/on-run-start/on-run-end). Let's look at a quick example:

```sql
-- models/foo.sql
{{ config(post_hook = "select 'success' as did_model_finish") }}
select 1 id
```

We have a straightforward hook that runs a SQL query (note that hooks CAN execute strings: https://gist.github.com/jeremyyeo/f97b6684643a9333d7901b4cefada32c).

```sh
$ dbt --debug run
03:57:24  1 of 1 START sql table model dev.foo ........................................... [RUN]
03:57:24  Re-using an available connection from the pool (formerly list_development_jyeo_dev, now model.my_dbt_project.foo)
03:57:24  Began compiling node model.my_dbt_project.foo
03:57:24  Writing injected SQL for node "model.my_dbt_project.foo"
03:57:24  Began executing node model.my_dbt_project.foo
03:57:24  Writing runtime sql for node "model.my_dbt_project.foo"
03:57:24  Using snowflake connection "model.my_dbt_project.foo"
03:57:24  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table development_jyeo.dev.foo
         as
        (
select 1 id
        );
03:57:24  Opening a new connection, currently in state closed
03:57:26  SQL status: SUCCESS 1 in 2.046 seconds
03:57:26  Using snowflake connection "model.my_dbt_project.foo"
03:57:26  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
select 'success' as did_model_finish
03:57:26  SQL status: SUCCESS 1 in 0.307 seconds
03:57:26  On model.my_dbt_project.foo: Close
03:57:27  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '445514e0-46dd-4033-b8df-2dfa78bf77f1', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x12078c410>]}
03:57:27  1 of 1 OK created sql table model dev.foo ...................................... [SUCCESS 1 in 3.02s]
```

^ Our model runs and then our hook runs after that - all good.

Some folks who are more advanced, may wrap that up in a macro...

```sql
-- models/foo.sql
{{ config(post_hook = "{{ do_something() }}") }}
select 1 id

-- macros/do_something.sql
{% macro do_something() %}
    select 'success' as did_model_finish
{% endmacro %}
```

^ Here, the macro `do_something()` returns a simple string (a string that so happen to be a SQL query text) to the hook. So it is as if it was exactly like things were previously.

```sh
$ dbt --debug run
03:59:28  1 of 1 START sql table model dev.foo ........................................... [RUN]
03:59:28  Re-using an available connection from the pool (formerly list_test_jyeo_dev, now model.my_dbt_project.foo)
03:59:28  Began compiling node model.my_dbt_project.foo
03:59:28  Writing injected SQL for node "model.my_dbt_project.foo"
03:59:28  Began executing node model.my_dbt_project.foo
03:59:28  Writing runtime sql for node "model.my_dbt_project.foo"
03:59:28  Using snowflake connection "model.my_dbt_project.foo"
03:59:28  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table development_jyeo.dev.foo
         as
        (
select 1 id
        );
03:59:28  Opening a new connection, currently in state closed
03:59:30  SQL status: SUCCESS 1 in 2.329 seconds
03:59:30  Using snowflake connection "model.my_dbt_project.foo"
03:59:30  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
select 'success' as did_model_finish
03:59:31  SQL status: SUCCESS 1 in 0.297 seconds
03:59:31  On model.my_dbt_project.foo: Close
03:59:31  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'a9782fa1-e85a-4e07-88c6-a500674fb143', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1184bf090>]}
03:59:31  1 of 1 OK created sql table model dev.foo ...................................... [SUCCESS 1 in 3.35s]
```

Now, folks who learn more about dbt, discover that there is this [`run_query()`](https://docs.getdbt.com/reference/dbt-jinja-functions/run_query) that they can use to run arbitrary SQL - then they may write the above like so:

```sql
-- models/foo.sql
{{ config(post_hook = "{{ do_something() }}") }}
select 1 id

-- macros/do_something.sql
{% macro do_something() %}
   {% do run_query("select 'success' as did_model_finish") %}
{% endmacro %}
```

```sh
$ dbt --debug run
04:03:36  1 of 1 START sql table model dev.foo ........................................... [RUN]
04:03:36  Re-using an available connection from the pool (formerly list_development_jyeo_dev, now model.my_dbt_project.foo)
04:03:36  Began compiling node model.my_dbt_project.foo
04:03:36  Writing injected SQL for node "model.my_dbt_project.foo"
04:03:36  Began executing node model.my_dbt_project.foo
04:03:36  Writing runtime sql for node "model.my_dbt_project.foo"
04:03:36  Using snowflake connection "model.my_dbt_project.foo"
04:03:36  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table development_jyeo.dev.foo
         as
        (
select 1 id
        );
04:03:36  Opening a new connection, currently in state closed
04:03:38  SQL status: SUCCESS 1 in 2.405 seconds
04:03:38  Using snowflake connection "model.my_dbt_project.foo"
04:03:38  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
select 'success' as did_model_finish
04:03:39  SQL status: SUCCESS 1 in 0.309 seconds
04:03:39  On model.my_dbt_project.foo: Close
04:03:39  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'c769e59c-f4d9-4fed-a821-cac0a28d01e2', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x109ac0390>]}
04:03:39  1 of 1 OK created sql table model dev.foo ...................................... [SUCCESS 1 in 3.38s]
```

Here, we get pretty much the exact same outcome as before. P.s. There is some additional complexity to when you want to use `run_query()` and when to not - but that is not the purpose of this lesson.

To take it one step further, a user may even add a comment to their macro because they want to explain to their colleague what they were doing:

```sql
-- models/foo.sql
{{ config(post_hook = "{{ do_something() }}") }}
select 1 id

-- macros/do_something.sql
{% macro do_something() %}
   -- I added this macro to track something.
   {% do run_query("select 'success' as did_model_finish") %}
{% endmacro %}
```

^ This seems like a very trivial addition to the macro but let's see what happens:

```sh
$ dbt --debug run
04:08:06  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table development_jyeo.dev.foo
         as
        (
select 1 id
        );
04:08:06  Opening a new connection, currently in state closed
04:08:08  SQL status: SUCCESS 1 in 2.320 seconds
04:08:08  Using snowflake connection "model.my_dbt_project.foo"
04:08:08  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
select 'success' as did_model_finish
04:08:08  SQL status: SUCCESS 1 in 0.313 seconds
04:08:08  Snowflake adapter: [WARNING]: Explicit transactional logic should be used only to wrap DML
logic (MERGE, DELETE, UPDATE, etc). The keywords BEGIN; and COMMIT; should be
placed directly before and after your DML statement, rather than in separate
statement calls or run_query() macros.
04:08:08  On model.my_dbt_project.foo: Close
04:08:09  Unhandled error while executing target/run/my_dbt_project/models/mart/foo.sql
cannot access local variable 'connection' where it is not associated with a value
04:08:09  Traceback (most recent call last):
<TRUNCATED>
    return connection, cursor
           ^^^^^^^^^^
UnboundLocalError: cannot access local variable 'connection' where it is not associated with a value

04:08:09  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'da526c59-eb2e-4967-9c1c-26f51ec1b32b', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x11df3e750>]}
04:08:09  1 of 1 ERROR creating sql table model dev.foo .................................. [ERROR in 3.34s]
```

^ Now, instead of running without errors, we get a kinda complicated warning/error instead. The reason for this is that we have effectively returned a SQL comment text to the hook and the Snowflake adapter will run into an error if you try to run a hook that is a SQL comment. We can quicky reproduce the problem without a macro, like so:

```sql
-- models/foo.sql
{{ config(post_hook = "-- I added this macro to track something.") }}
select 1 id
```

```sh
$ dbt --debug run
04:11:53  1 of 1 START sql table model dev.foo ........................................... [RUN]
04:11:53  Re-using an available connection from the pool (formerly list_development_jyeo_dev, now model.my_dbt_project.foo)
04:11:53  Began compiling node model.my_dbt_project.foo
04:11:53  Writing injected SQL for node "model.my_dbt_project.foo"
04:11:53  Began executing node model.my_dbt_project.foo
04:11:53  Writing runtime sql for node "model.my_dbt_project.foo"
04:11:53  Using snowflake connection "model.my_dbt_project.foo"
04:11:53  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table development_jyeo.dev.foo
         as
        (
select 1 id
        );
04:11:53  Opening a new connection, currently in state closed
04:11:55  SQL status: SUCCESS 1 in 2.561 seconds
04:11:55  Snowflake adapter: [WARNING]: Explicit transactional logic should be used only to wrap DML
logic (MERGE, DELETE, UPDATE, etc). The keywords BEGIN; and COMMIT; should be
placed directly before and after your DML statement, rather than in separate
statement calls or run_query() macros.
04:11:55  On model.my_dbt_project.foo: Close
04:11:56  Unhandled error while executing target/run/my_dbt_project/models/mart/foo.sql
cannot access local variable 'connection' where it is not associated with a value
04:11:56  Traceback (most recent call last):
<TRUNCATED>
    return connection, cursor
           ^^^^^^^^^^
UnboundLocalError: cannot access local variable 'connection' where it is not associated with a value

04:11:56  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'c487550b-9e95-46db-94fe-980d92c06a99', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1175ceb90>]}
04:11:56  1 of 1 ERROR creating sql table model dev.foo .................................. [ERROR in 3.26s]
```

It's not possible for a hook to successfully run a hook when the hook is simply a SQL comment text.

So if you're running into a warning/error like this, be sure to double check that you're not returning something invalid like a SQL comment to your hook and if you must add comments - do something like:

```sql
-- macros/do_something.sql
{% macro do_something() %}
   {#/* I added this macro to track something. */#}
   {% do run_query("select 'success' as did_model_finish") %}
{% endmacro %}
```

Because that SQL comment is inside a Jinja comment (delimited by `{#` and `#}`) - then that won't get returned to the caller in the hook.

### Found patch for macro "..." which was not found

This can happen when we document a macro which doesn't exist:

```yaml
# dbt_project.yml
name: analytics
profile: sf
version: "1.0.0"

models:
  analytics:
    +materialized: table

# macros/schema.yml
macros:
  - name: cents_to_dollars
    description: A macro to convert cents to dollars
```

```sql
-- models/foo.sql
select 1 id
```

```sh
$ dbt parse --no-partial-parse
04:43:56  Running with dbt=1.10.2
04:43:56  Registered adapter: snowflake=1.9.4
04:43:57  [WARNING]: Found patch for macro "cents_to_dollars" which was not found
04:43:57  Performance info: /Users/jeremy/git/dbt-basic/target/perf_info.json
```

We have no macro named `cents_to_dollars` in our project, but we have a `schema.yml` that attempted to describe it.

### CustomKeyInConfigDeprecation

https://docs.getdbt.com/reference/deprecations?version=2.0#customkeyinconfigdeprecation

Historically, dbt allowed users to set "custom configs" (i.e. non-standard configs like `materialized`, `schema`) at the root level - however, this caused a lot of validation issues (e.g. users typing `materialization='table'` instead of `materialized='view'` - this raised no errors therefore it was highly likely dbt would silently just build those models as views and users were unaware).

To test this, we're going to use a similar custom materialization as shown [here](../custom-materialization)

```sql
-- macros/custom_table.sql
{% materialization custom_table, adapter='snowflake' %}
    {%- set tenants = config.get('tenants') -%}
    {%- set language = model['language'] -%}
    {%- set target_relation = api.Relation.create(identifier=model['name'], schema=schema, database=database, type='table') -%}

    {%- for tenant in tenants -%}
        {%- set identifier = model['name'] ~ '_' ~ tenant -%}
        {%- set existing_relation = adapter.get_relation(database=database, schema=schema, identifier=identifier) -%}
        {%- set target_relation = api.Relation.create(identifier=identifier, schema=schema, database=database, type='table') -%}
        {% call statement('main') -%}
           create or replace table {{ target_relation }} as ({{- sql -}});
        {%- endcall %}
    {%- endfor -%}

    {{ return({'relations': [target_relation]}) }}
{% endmaterialization %}

-- models/foo.sql
{{-
    config(
        materialized='custom_table',
        tenants=['dunder_miffline', 'central_perk']
    )
-}}

select 100 as sales
```

```sh
$ dbt --debug run --no-partial-parse
...
21:47:35  1 of 1 START sql custom_table model sch.foo .................................... [RUN]
21:47:35  Re-using an available connection from the pool (formerly list_db_sch, now model.dbt_basic_5.foo)
21:47:35  Began compiling node model.dbt_basic_5.foo
21:47:35  Writing injected SQL for node "model.dbt_basic_5.foo"
21:47:35  Began executing node model.dbt_basic_5.foo
21:47:35  Writing runtime sql for node "model.dbt_basic_5.foo"
21:47:35  Using snowflake connection "model.dbt_basic_5.foo"
21:47:35  On model.dbt_basic_5.foo: create or replace table db.sch.foo_dunder_miffline as (select 100 as sales)
/* {"app": "dbt", "dbt_version": "1.11.8", "profile_name": "sf", "target_name": "ci", "node_id": "model.dbt_basic_5.foo"} */;
21:47:36  SQL status: SUCCESS 1 in 0.855 seconds
21:47:36  Writing runtime sql for node "model.dbt_basic_5.foo"
21:47:36  Using snowflake connection "model.dbt_basic_5.foo"
21:47:36  On model.dbt_basic_5.foo: create or replace table db.sch.foo_central_perk as (select 100 as sales)
/* {"app": "dbt", "dbt_version": "1.11.8", "profile_name": "sf", "target_name": "ci", "node_id": "model.dbt_basic_5.foo"} */;
21:47:37  SQL status: SUCCESS 1 in 0.627 seconds
21:47:37  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'c75f8e6b-eb9d-4a70-ad4e-25bb6054afd8', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x12771bbd0>]}
21:47:37  1 of 1 OK created sql custom_table model sch.foo ............................... [SUCCESS 1 in 1.51s]
21:47:37  Finished running node model.dbt_basic_5.foo
21:47:37  Connection 'master' was properly closed.
21:47:37  Connection 'model.dbt_basic_5.foo' was left open.
21:47:37  On model.dbt_basic_5.foo: Close
21:47:37
21:47:37  Finished running 1 custom table model in 0 hours 0 minutes and 2.59 seconds (2.59s).
21:47:37  Command end result
21:47:37  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
21:47:37  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
21:47:37  Wrote artifact RunExecutionResult to /Users/jeremy/git/dbt-basic/target/run_results.json
21:47:37
21:47:37  Completed successfully
21:47:37
21:47:37  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=1
21:47:37  [WARNING][DeprecationsSummary]: Deprecated functionality
Summary of encountered deprecations:
- CustomKeyInConfigDeprecation: 1 occurrence
To see all deprecation instances instead of just the first occurrence of each,
run command again with the `--show-all-deprecations` flag. You may also need to
run with `--no-partial-parse` as some deprecations are only encountered during
parsing.
```

This worked as expected but with the warning thrown... let's try to get it compliant (no warnings):

```diff
#-- macros/custom_table.sql

{% materialization custom_table, adapter='snowflake' %}
-   {%- set tenants = config.get('tenants') -%}
+   {%- set tenants = config.get('meta')['tenants'] -%}
    {%- set language = model['language'] -%}
    {%- set target_relation = api.Relation.create(identifier=model['name'], schema=schema, database=database, type='table') -%}

    {%- for tenant in tenants -%}
        {%- set identifier = model['name'] ~ '_' ~ tenant -%}
        {%- set existing_relation = adapter.get_relation(database=database, schema=schema, identifier=identifier) -%}
        {%- set target_relation = api.Relation.create(identifier=identifier, schema=schema, database=database, type='table') -%}
        {% call statement('main') -%}
           create or replace table {{ target_relation }} as ({{- sql -}});
        {%- endcall %}
    {%- endfor -%}

    {{ return({'relations': [target_relation]}) }}
{% endmaterialization %}
```

^ As we can see here, instead of "getting" `tenants` directly from `config`, we would simply now retrieve it from the `config.meta` key instead. Note: doing `tenants = config.meta_get('tenants')` would be another alternative to the approach shown above.

```diff
#-- models/foo.sql
{{
    config(
        materialized='custom_table',
-       tenants=['dunder_miffline', 'central_perk']
+       meta={'tenants':['dunder_miffline', 'central_perk']}
    )
}}

select 100 as sales
```

```sh
$ dbt --debug run --no-partial-parse
...
21:54:45  1 of 1 START sql custom_table model sch.foo .................................... [RUN]
21:54:45  Re-using an available connection from the pool (formerly list_db_sch, now model.dbt_basic_5.foo)
21:54:45  Began compiling node model.dbt_basic_5.foo
21:54:45  Writing injected SQL for node "model.dbt_basic_5.foo"
21:54:45  Began executing node model.dbt_basic_5.foo
21:54:45  Writing runtime sql for node "model.dbt_basic_5.foo"
21:54:45  Using snowflake connection "model.dbt_basic_5.foo"
21:54:45  On model.dbt_basic_5.foo: create or replace table db.sch.foo_dunder_miffline as (select 100 as sales)
/* {"app": "dbt", "dbt_version": "1.11.8", "profile_name": "sf", "target_name": "ci", "node_id": "model.dbt_basic_5.foo"} */;
21:54:46  SQL status: SUCCESS 1 in 1.213 seconds
21:54:46  Writing runtime sql for node "model.dbt_basic_5.foo"
21:54:46  Using snowflake connection "model.dbt_basic_5.foo"
21:54:46  On model.dbt_basic_5.foo: create or replace table db.sch.foo_central_perk as (select 100 as sales)
/* {"app": "dbt", "dbt_version": "1.11.8", "profile_name": "sf", "target_name": "ci", "node_id": "model.dbt_basic_5.foo"} */;
21:54:48  SQL status: SUCCESS 1 in 1.402 seconds
21:54:48  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'c9461ab7-81bc-4f8f-9c93-e19fba2bf726', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1221df210>]}
21:54:48  1 of 1 OK created sql custom_table model sch.foo ............................... [SUCCESS 1 in 2.65s]
21:54:48  Finished running node model.dbt_basic_5.foo
21:54:48  Connection 'master' was properly closed.
21:54:48  Connection 'model.dbt_basic_5.foo' was left open.
21:54:48  On model.dbt_basic_5.foo: Close
21:54:48
21:54:48  Finished running 1 custom table model in 0 hours 0 minutes and 3.64 seconds (3.64s).
21:54:48  Command end result
21:54:48  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
21:54:48  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
21:54:48  Wrote artifact RunExecutionResult to /Users/jeremy/git/dbt-basic/target/run_results.json
21:54:48
21:54:48  Completed successfully
21:54:48
21:54:48  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=1

$ dbtf run --log-level debug
...
Started rendering model sch.foo
Finished rendering [  0.00s] model sch.foo [success]
Started analyzing model sch.foo
Finished analyzing [  0.01s] model sch.foo [success]
Started running model sch.foo
Query executed on node model.dbt_basic_5.foo:
SHOW OBJECTS IN SCHEMA "DB"."SCH" LIMIT 10000
Writing runtime sql for node "model.dbt_basic_5.foo"
Query executed on node model.dbt_basic_5.foo:
create or replace table db.sch.foo_dunder_miffline as (

select 100 as sales)
/* {"app": "dbt", "dbt_version": "2.0.0", "node_id": "model.dbt_basic_5.foo", "profile_name": "sf", "target_name": "ci"} */
Writing runtime sql for node "model.dbt_basic_5.foo"
Query executed on node model.dbt_basic_5.foo:
create or replace table db.sch.foo_central_perk as (

select 100 as sales)
/* {"app": "dbt", "dbt_version": "2.0.0", "node_id": "model.dbt_basic_5.foo", "profile_name": "sf", "target_name": "ci"} */
 Succeeded [  3.31s] model sch.foo (custom_table)
Finished running [  3.30s] model sch.foo [success]
```

We're also going to do some additional testing to see if the new nesting of "custom configs" under meta works for some other places:

#### In a model's `schema.yml` file

```sql
-- models/foo.sql
select 100 as sales
```

```yaml
# dbt_project.yml
name: dbt_basic_5
profile: sf
version: "1.0.0"

# models/schema.yml
models:
  - name: foo
    config:
      materialized: custom_table
      meta:
        tenants:
          - dunder_mifflin
          - central_perk
```

```sh
$ dbt --debug run --no-partial-parse
...
22:00:09  1 of 1 START sql custom_table model sch.foo .................................... [RUN]
22:00:09  Re-using an available connection from the pool (formerly list_db_sch, now model.dbt_basic_5.foo)
22:00:09  Began compiling node model.dbt_basic_5.foo
22:00:09  Writing injected SQL for node "model.dbt_basic_5.foo"
22:00:09  Began executing node model.dbt_basic_5.foo
22:00:09  Writing runtime sql for node "model.dbt_basic_5.foo"
22:00:09  Using snowflake connection "model.dbt_basic_5.foo"
22:00:09  On model.dbt_basic_5.foo: create or replace table db.sch.foo_dunder_mifflin as (select 100 as sales)
/* {"app": "dbt", "dbt_version": "1.11.8", "profile_name": "sf", "target_name": "ci", "node_id": "model.dbt_basic_5.foo"} */;
22:00:10  SQL status: SUCCESS 1 in 1.440 seconds
22:00:10  Writing runtime sql for node "model.dbt_basic_5.foo"
22:00:10  Using snowflake connection "model.dbt_basic_5.foo"
22:00:10  On model.dbt_basic_5.foo: create or replace table db.sch.foo_central_perk as (select 100 as sales)
/* {"app": "dbt", "dbt_version": "1.11.8", "profile_name": "sf", "target_name": "ci", "node_id": "model.dbt_basic_5.foo"} */;
22:00:12  SQL status: SUCCESS 1 in 1.931 seconds
22:00:12  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'bb35d3ae-1308-44f5-afe3-c24a8872f101', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x12663f0d0>]}
22:00:12  1 of 1 OK created sql custom_table model sch.foo ............................... [SUCCESS 1 in 3.40s]
22:00:12  Finished running node model.dbt_basic_5.foo
22:00:12  Connection 'master' was properly closed.
22:00:12  Connection 'model.dbt_basic_5.foo' was left open.
22:00:12  On model.dbt_basic_5.foo: Close
22:00:12
22:00:12  Finished running 1 custom table model in 0 hours 0 minutes and 5.23 seconds (5.23s).
22:00:12  Command end result
22:00:12  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
22:00:12  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
22:00:12  Wrote artifact RunExecutionResult to /Users/jeremy/git/dbt-basic/target/run_results.json
22:00:12
22:00:12  Completed successfully
22:00:12
22:00:12  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=1

$ dbtf run --log-level debug
...
Started rendering model sch.foo
Finished rendering [  0.00s] model sch.foo [success]
Started analyzing model sch.foo
Finished analyzing [  0.01s] model sch.foo [success]
Started running model sch.foo
Query executed on node model.dbt_basic_5.foo:
SHOW OBJECTS IN SCHEMA "DB"."SCH" LIMIT 10000
Writing runtime sql for node "model.dbt_basic_5.foo"
Query executed on node model.dbt_basic_5.foo:
create or replace table db.sch.foo_dunder_mifflin as (
select 100 as sales)
/* {"app": "dbt", "dbt_version": "2.0.0", "node_id": "model.dbt_basic_5.foo", "profile_name": "sf", "target_name": "ci"} */
Writing runtime sql for node "model.dbt_basic_5.foo"
Query executed on node model.dbt_basic_5.foo:
create or replace table db.sch.foo_central_perk as (
select 100 as sales)
/* {"app": "dbt", "dbt_version": "2.0.0", "node_id": "model.dbt_basic_5.foo", "profile_name": "sf", "target_name": "ci"} */
 Succeeded [  2.90s] model sch.foo (custom_table)
Finished running [  2.89s] model sch.foo [success]
```

#### In the project level `dbt_project.yml` file

```sql
-- models/foo.sql
select 100 as sales
```

```yaml
# dbt_project.yml
name: dbt_basic_5
profile: sf
version: "1.0.0"

models:
  dbt_basic_5:
    +materialized: custom_table
    +meta:
      tenants:
        - dunder_mifflin
        - stark_industries
```

```sh
$ dbt --debug run --no-partial-parse
...
22:03:57  1 of 1 START sql custom_table model sch.foo .................................... [RUN]
22:03:57  Re-using an available connection from the pool (formerly list_db_sch, now model.dbt_basic_5.foo)
22:03:57  Began compiling node model.dbt_basic_5.foo
22:03:57  Writing injected SQL for node "model.dbt_basic_5.foo"
22:03:57  Began executing node model.dbt_basic_5.foo
22:03:57  Writing runtime sql for node "model.dbt_basic_5.foo"
22:03:57  Using snowflake connection "model.dbt_basic_5.foo"
22:03:57  On model.dbt_basic_5.foo: create or replace table db.sch.foo_dunder_mifflin as (select 100 as sales)
/* {"app": "dbt", "dbt_version": "1.11.8", "profile_name": "sf", "target_name": "ci", "node_id": "model.dbt_basic_5.foo"} */;
22:03:58  SQL status: SUCCESS 1 in 1.123 seconds
22:03:58  Writing runtime sql for node "model.dbt_basic_5.foo"
22:03:58  Using snowflake connection "model.dbt_basic_5.foo"
22:03:58  On model.dbt_basic_5.foo: create or replace table db.sch.foo_stark_industries as (select 100 as sales)
/* {"app": "dbt", "dbt_version": "1.11.8", "profile_name": "sf", "target_name": "ci", "node_id": "model.dbt_basic_5.foo"} */;
22:03:59  SQL status: SUCCESS 1 in 0.910 seconds
22:03:59  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '8d74e8ab-94a6-499f-bd52-a9bfb0e53f12', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1252fc8d0>]}
22:03:59  1 of 1 OK created sql custom_table model sch.foo ............................... [SUCCESS 1 in 2.07s]
22:03:59  Finished running node model.dbt_basic_5.foo
22:03:59  Connection 'master' was properly closed.
22:03:59  Connection 'model.dbt_basic_5.foo' was left open.
22:03:59  On model.dbt_basic_5.foo: Close
22:03:59
22:03:59  Finished running 1 custom table model in 0 hours 0 minutes and 3.30 seconds (3.30s).
22:03:59  Command end result
22:03:59  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
22:03:59  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
22:03:59  Wrote artifact RunExecutionResult to /Users/jeremy/git/dbt-basic/target/run_results.json
22:03:59
22:03:59  Completed successfully
22:03:59
22:03:59  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=1

$ dbtf run --log-level debug
...
Started rendering model sch.foo
Finished rendering [  0.00s] model sch.foo [success]
Started analyzing model sch.foo
Finished analyzing [  0.00s] model sch.foo [success]
Started running model sch.foo
Query executed on node model.dbt_basic_5.foo:
SHOW OBJECTS IN SCHEMA "DB"."SCH" LIMIT 10000
Writing runtime sql for node "model.dbt_basic_5.foo"
Query executed on node model.dbt_basic_5.foo:
create or replace table db.sch.foo_dunder_mifflin as (
select 100 as sales)
/* {"app": "dbt", "dbt_version": "2.0.0", "node_id": "model.dbt_basic_5.foo", "profile_name": "sf", "target_name": "ci"} */
Writing runtime sql for node "model.dbt_basic_5.foo"
Query executed on node model.dbt_basic_5.foo:
create or replace table db.sch.foo_stark_industries as (
select 100 as sales)
/* {"app": "dbt", "dbt_version": "2.0.0", "node_id": "model.dbt_basic_5.foo", "profile_name": "sf", "target_name": "ci"} */
 Succeeded [  3.43s] model sch.foo (custom_table)
Finished running [  3.43s] model sch.foo [success]
```
