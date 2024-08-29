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
