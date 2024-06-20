---
---

## Dynamically changing the warehouse used for incremental models

> Full credits to user `dbernett110` on the dbt Community Slack.

Watch this: https://youtu.be/oVsqvHOBdgs

A common ask of dbt is to conditionally switch to a different warehouse (other than the one specified by default) for an incremental model if it is being created for the first time. An incremental model will be created from scratch (i.e. the ddl is `create or replace table as ...` instead of `insert ...` or `merge ...`) if:
* The model did not exist yet (i.e. the `is_incremental()` check evaluated to `False`).
* The invocation had the `--full-refresh` flag.

One typically tries to approach this by adding some logic into the `snowflake_warehouse` config for models like so:

```sql
-- models/my_inc.sql
{{ 
    config(
        materialized = 'incremental',
        snowflake_warehouse = ('wh_large' if is_incremental() else target.warehouse),
        ...
    )
}}

-- models/my_inc.sql
{{ 
    config(
        materialized = 'incremental',
        snowflake_warehouse = ('wh_large' if flags.FULL_REFRESH else target.warehouse),
        ...
    )
}}
```

> Or some variation of the examples shown above.

However, the first method doesn't quite work because dbt sets model configs prior to doing the `is_incremental()` check which results in issues like https://github.com/dbt-labs/dbt-snowflake/issues/103 and the second method doesn't work if [partial parsing](https://docs.getdbt.com/reference/parsing#partial-parsing) is turned on.

## Workaround

In this exercise, we want to achieve the following:
* If an incremental model is being built for the first time, i.e. `not is_incremental()` or `flags.FULL_REFRESH`, then use the larger warehouse (`wh_large`) for the `create or replace table as ...` statement
* If it is not being built for the first time, then use the default warehouse on the connection (`wh_small`) for the `insert ...` or `merge ...` statement.
* We also expect this to work even with partial parsing turned on and we're going back and forth between turning on and off the `--full-refresh` flag.

First, we setup our `profile.yml` with the default warehouse to use (`wh_small`):

```yml
# ~/.dbt/profiles.yml
snowflake:
  target: default
  outputs:
    default:
      type: snowflake
      warehouse: wh_small
      ...
```

> In dbt Cloud - you would use the various forms and fields to configure your default warehouse as dbt Cloud does not use a `profiles.yml` file.

And we add a macro that will execute a `use warehouse wh_large` statement in order to switch to the larger warehouse:

```sql
-- macros/use_wh.sql
{% macro use_wh() %}
    {% if execute %}
        {% do run_query('use warehouse wh_large;') %}
    {% endif %}
{% endmacro %}
```

In our incremental model, we will call that macro on both the conditions where we want the larger warehouse to be used:

```sql
-- models/my_inc.sql
{{ config(materialized='incremental') }}

{% if not is_incremental() or flags.FULL_REFRESH %}
    {{ use_wh() }}
{% endif %}

select 1 id
```

## Testing

Let's first make sure the relation `my_inc` doesn't already exist by dropping it from Snowflake so we're starting from the very beginning:

```sql
drop table if exists development_jyeo.dbt_jyeo.my_inc
```

Build `my_inc` for the first time and check the debug logs + Snowflake query history:

<details>
  <summary>Debug logs</summary>

```sh
$ dbt --debug build
00:07:32  Began running node model.my_dbt_project.my_inc
00:07:32  1 of 1 START sql incremental model dbt_jyeo.my_inc ............................. [RUN]
00:07:32  Re-using an available connection from the pool (formerly list_development_jyeo_dbt_jyeo, now model.my_dbt_project.my_inc)
00:07:32  Began compiling node model.my_dbt_project.my_inc
00:07:32  Using snowflake connection "model.my_dbt_project.my_inc"
00:07:32  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
use warehouse wh_large;
00:07:32  Opening a new connection, currently in state closed
00:07:34  SQL status: SUCCESS 1 in 2.0 seconds
00:07:34  Writing injected SQL for node "model.my_dbt_project.my_inc"
00:07:34  Began executing node model.my_dbt_project.my_inc
00:07:34  Writing runtime sql for node "model.my_dbt_project.my_inc"
00:07:34  Using snowflake connection "model.my_dbt_project.my_inc"
00:07:34  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
create or replace transient table development_jyeo.dbt_jyeo.my_inc
         as
        (
select 1 id
        );
00:07:35  SQL status: SUCCESS 1 in 1.0 seconds
00:07:35  Applying DROP to: development_jyeo.dbt_jyeo.my_inc__dbt_tmp
00:07:35  Using snowflake connection "model.my_dbt_project.my_inc"
00:07:35  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
drop view if exists development_jyeo.dbt_jyeo.my_inc__dbt_tmp cascade
00:07:35  SQL status: SUCCESS 1 in 0.0 seconds
00:07:35  On model.my_dbt_project.my_inc: Close
00:07:36  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '14acc6a5-8592-406e-8dd7-b12804ae6eab', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x126a3b090>]}
00:07:36  1 of 1 OK created sql incremental model dbt_jyeo.my_inc ........................ [SUCCESS 1 in 3.88s]
```

</details>

```sh
# snowsql output of checking the query history
╒═══════════════════════════════╤════════════════╤═══════════════════════════════════════════════════════════════════════╕
│ START_TIME                    │ WAREHOUSE_NAME │ QUERY_TEXT                                                            │
╞═══════════════════════════════╪════════════════╪═══════════════════════════════════════════════════════════════════════╡
│ 2024-05-24 00:07:35.925 +0000 │ WH_LARGE       │ drop view if exists development_jyeo.dbt_jyeo.my_inc__dbt_tmp cascade │
├───────────────────────────────┼────────────────┼───────────────────────────────────────────────────────────────────────┤
│ 2024-05-24 00:07:34.731 +0000 │ WH_LARGE       │ create or replace transient table development_jyeo.dbt_jyeo.my_inc    │
│                               │                │          as                                                           │
│                               │                │         (                                                             │
│                               │                │ select 1 id                                                           │
│                               │                │         );                                                            │
├───────────────────────────────┼────────────────┼───────────────────────────────────────────────────────────────────────┤
│ 2024-05-24 00:07:34.340 +0000 │ WH_SMALL       │ use warehouse wh_large;                                               │
╘═══════════════════════════════╧════════════════╧═══════════════════════════════════════════════════════════════════════╛
```

The intial run works as expected and the `create or replace ...` statement used the `wh_large` warehouse as expected.

Let's do our next subsequent run:

<details>
  <summary>Debug logs</summary>

```sh
$ dbt --debug build
00:11:22  1 of 1 START sql incremental model dbt_jyeo.my_inc ............................. [RUN]
00:11:22  Re-using an available connection from the pool (formerly list_development_jyeo, now model.my_dbt_project.my_inc)
00:11:22  Began compiling node model.my_dbt_project.my_inc
00:11:22  Writing injected SQL for node "model.my_dbt_project.my_inc"
00:11:22  Began executing node model.my_dbt_project.my_inc
00:11:22  Using snowflake connection "model.my_dbt_project.my_inc"
00:11:22  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
create or replace  temporary view development_jyeo.dbt_jyeo.my_inc__dbt_tmp
   as (
select 1 id
  );
00:11:22  Opening a new connection, currently in state closed
00:11:23  SQL status: SUCCESS 1 in 2.0 seconds
00:11:23  Using snowflake connection "model.my_dbt_project.my_inc"
00:11:23  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
describe table development_jyeo.dbt_jyeo.my_inc__dbt_tmp
00:11:24  SQL status: SUCCESS 1 in 0.0 seconds
00:11:24  Using snowflake connection "model.my_dbt_project.my_inc"
00:11:24  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
describe table development_jyeo.dbt_jyeo.my_inc
00:11:24  SQL status: SUCCESS 1 in 0.0 seconds
00:11:24  Using snowflake connection "model.my_dbt_project.my_inc"
00:11:24  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
describe table "DEVELOPMENT_JYEO"."DBT_JYEO"."MY_INC"
00:11:24  SQL status: SUCCESS 1 in 0.0 seconds
00:11:24  Writing runtime sql for node "model.my_dbt_project.my_inc"
00:11:24  Using snowflake connection "model.my_dbt_project.my_inc"
00:11:24  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
-- back compat for old kwarg name
  
  begin;
00:11:25  SQL status: SUCCESS 1 in 0.0 seconds
00:11:25  Using snowflake connection "model.my_dbt_project.my_inc"
00:11:25  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
insert into development_jyeo.dbt_jyeo.my_inc ("ID")
        (
            select "ID"
            from development_jyeo.dbt_jyeo.my_inc__dbt_tmp
        );
00:11:26  SQL status: SUCCESS 1 in 1.0 seconds
00:11:26  Using snowflake connection "model.my_dbt_project.my_inc"
00:11:26  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
COMMIT
00:11:26  SQL status: SUCCESS 1 in 0.0 seconds
00:11:26  Applying DROP to: development_jyeo.dbt_jyeo.my_inc__dbt_tmp
00:11:26  Using snowflake connection "model.my_dbt_project.my_inc"
00:11:26  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
drop view if exists development_jyeo.dbt_jyeo.my_inc__dbt_tmp cascade
00:11:27  SQL status: SUCCESS 1 in 0.0 seconds
00:11:27  On model.my_dbt_project.my_inc: Close
00:11:28  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'd2b39c15-7e5a-4157-8676-6473d09449a2', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x102de7410>]}
00:11:28  1 of 1 OK created sql incremental model dbt_jyeo.my_inc ........................ [SUCCESS 1 in 5.84s]
```

</details>


```sh
# snowsql output of checking the query history
╒═══════════════════════════════╤════════════════╤═════════════════════════════════════════════════════════════════════════════╕
│ START_TIME                    │ WAREHOUSE_NAME │ QUERY_TEXT                                                                  │
╞═══════════════════════════════╪════════════════╪═════════════════════════════════════════════════════════════════════════════╡
│ 2024-05-24 00:11:27.337 +0000 │ WH_SMALL       │ drop view if exists development_jyeo.dbt_jyeo.my_inc__dbt_tmp cascade       │
├───────────────────────────────┼────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ 2024-05-24 00:11:25.724 +0000 │ WH_SMALL       │ insert into development_jyeo.dbt_jyeo.my_inc ("ID")                         │
│                               │                │         (                                                                   │
│                               │                │             select "ID"                                                     │
│                               │                │             from development_jyeo.dbt_jyeo.my_inc__dbt_tmp                  │
│                               │                │         );                                                                  │
├───────────────────────────────┼────────────────┼─────────────────────────────────────────────────────────────────────────────┤
│ 2024-05-24 00:11:23.876 +0000 │ WH_SMALL       │ create or replace  temporary view development_jyeo.dbt_jyeo.my_inc__dbt_tmp │
│                               │                │    as (                                                                     │
│                               │                │ select 1 id                                                                 │
│                               │                │   );                                                                        │
╘═══════════════════════════════╧════════════════╧═════════════════════════════════════════════════════════════════════════════╛
```

Since this is the subsequent incremental run, dbt issued an `insert into` instead and we see that it was using the smaller `wh_small` warehouse as expected.

Now let's try to build this table from scratch via building using the `--full-fresh`:

<details>
  <summary>Debug logs</summary>

```sh
$ dbt --debug build
...
00:18:48  Partial parsing enabled: 0 files deleted, 0 files added, 0 files changed.
00:18:48  Partial parsing enabled, no changes found, skipping parsing
...
00:18:54  1 of 1 START sql incremental model dbt_jyeo.my_inc ............................. [RUN]
00:18:54  Re-using an available connection from the pool (formerly list_development_jyeo_dbt_jyeo, now model.my_dbt_project.my_inc)
00:18:54  Began compiling node model.my_dbt_project.my_inc
00:18:54  Using snowflake connection "model.my_dbt_project.my_inc"
00:18:54  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
use warehouse wh_large;
00:18:54  Opening a new connection, currently in state closed
00:18:56  SQL status: SUCCESS 1 in 2.0 seconds
00:18:56  Writing injected SQL for node "model.my_dbt_project.my_inc"
00:18:56  Began executing node model.my_dbt_project.my_inc
00:18:56  Writing runtime sql for node "model.my_dbt_project.my_inc"
00:18:56  Using snowflake connection "model.my_dbt_project.my_inc"
00:18:56  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
create or replace transient table development_jyeo.dbt_jyeo.my_inc
         as
        (
select 1 id
        );
00:18:57  SQL status: SUCCESS 1 in 1.0 seconds
00:18:57  Applying DROP to: development_jyeo.dbt_jyeo.my_inc__dbt_tmp
00:18:57  Using snowflake connection "model.my_dbt_project.my_inc"
00:18:57  On model.my_dbt_project.my_inc: /* {"app": "dbt", "dbt_version": "1.8.1", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.my_inc"} */
drop view if exists development_jyeo.dbt_jyeo.my_inc__dbt_tmp cascade
00:18:58  SQL status: SUCCESS 1 in 0.0 seconds
00:18:58  On model.my_dbt_project.my_inc: Close
00:18:58  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'c7a3b070-1eaf-40f8-951c-57f14bda631d', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x111198510>]}
00:18:58  1 of 1 OK created sql incremental model dbt_jyeo.my_inc ........................ [SUCCESS 1 in 3.93s]
```

</details>

> Note here that partial parsing has been on for all of these invocations and things have been working as expected. If we tried to do this by adding logic to the `snowflake_warehouse` config - we would see that the `create or replace ...` statement use the `wh_small` warehouse instead.

```sh
# snowsql output of checking the query history
╒═══════════════════════════════╤════════════════╤═══════════════════════════════════════════════════════════════════════╕
│ START_TIME                    │ WAREHOUSE_NAME │ QUERY_TEXT                                                            │
╞═══════════════════════════════╪════════════════╪═══════════════════════════════════════════════════════════════════════╡
│ 2024-05-24 00:18:58.085 +0000 │ WH_LARGE       │ drop view if exists development_jyeo.dbt_jyeo.my_inc__dbt_tmp cascade │
├───────────────────────────────┼────────────────┼───────────────────────────────────────────────────────────────────────┤
│ 2024-05-24 00:18:56.999 +0000 │ WH_LARGE       │ create or replace transient table development_jyeo.dbt_jyeo.my_inc    │
│                               │                │          as                                                           │
│                               │                │         (                                                             │
│                               │                │ select 1 id                                                           │
│                               │                │         );                                                            │
├───────────────────────────────┼────────────────┼───────────────────────────────────────────────────────────────────────┤
│ 2024-05-24 00:18:56.607 +0000 │ WH_SMALL       │ use warehouse wh_large;                                               │
╘═══════════════════════════════╧════════════════╧═══════════════════════════════════════════════════════════════════════╛
```

The `--full-refresh` flag causes the model to be build from scratch and we do indeed see once again that the right warehouse `wh_large` is used.
