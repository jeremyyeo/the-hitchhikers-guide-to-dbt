---
---

## Incremental microbatch models

1. [The default behaviour of microbatch models](#default-behaviour)
2. [Dynamic begin config based on environment](#dynamically-limiting-the-begin-config-depending-on-the-environment--target)

### Default behaviour

This is a quick write up on the default behaviour of microbatch models - with various scenarios. We're going to test by incrementing our system date, one day at a time to simulate the fact that we're running our job (i.e. dbt) on a different date/day.

TL'DR:

1. The checkpoint / start date / the first batch is the current time period minus 1. Example: for a batch size of `day`, if dbt is running today and the current date today is `2025-03-05`, then the first batch run would be `2023-03-04`.
2. The start date / first batch can be further in the past by configuring a static `lookback` config (default is `1` which gives the above behaviour).

#### Day 1 (2025-03-01)

The date of `2025-03-01` is the very start of our project.

First let's create the raw source we're going to be selecting from and using in our microbatch model:

```sql
create or replace table db.raw.customers as (select 1 as id, 'alice' as first_name, '2025-03-01'::date as loaded_at);
```

Which we then define as a source in dbt:

```yml
# models/sources.yml
sources:
  - name: raw
    tables:
      - name: customers
        config:
          event_time: loaded_at
```

Which we select in an incremental microbatch model:

```sql
-- models/my_first_microbatch.sql
{{ 
    config(
        materialized='incremental',
        unique_key='id',
        incremental_strategy='microbatch',
        event_time='updated_at',
        begin='2025-03-01',
        batch_size='day',
        concurrent_batches=False
    ) 
}}

select id, first_name, loaded_at as updated_at from {{ source('raw', 'customers') }}
```

First build:

```sh
$ sudo date -u 0301000025 # Set system datetime to 2025-03-20 00:00 UTC on macOS
$ dbt build

[0m13:04:49.186344 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:04:49.187027 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:04:49.187825 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:04:49.188445 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:04:49.191311 [debug] [Thread-1 (]: batch 2025-03-01 of sch.my_first_microbatch is being run sequentially
[0m13:04:49.191940 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:04:49.192711 [info ] [Thread-1 (]: Batch 1 of 1 START batch 2025-03-01 of sch.my_first_microbatch ....................... [RUN]
[0m13:04:49.193400 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:04:49.204735 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:04:49.205988 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:04:49.282633 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:04:49.284317 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:04:49.285020 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace transient table db.sch.my_first_microbatch
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-01 00:00:00+00:00' and loaded_at < '2025-03-02 00:00:00+00:00')
        );
[0m13:04:50.431311 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.145 seconds
[0m13:04:50.453606 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250301
[0m13:04:50.464963 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:04:50.465668 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250301 cascade
[0m13:04:50.697857 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.231 seconds
[0m13:04:50.739933 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '5cc46d72-bb5f-44b1-a27e-bafc9655882f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x11158ad10>]}
[0m13:04:50.741207 [info ] [Thread-1 (]: Batch 1 of 1 OK created batch 2025-03-01 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 1.55s]
[0m13:04:50.742547 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:04:50.743902 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '5cc46d72-bb5f-44b1-a27e-bafc9655882f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1100b29d0>]}
[0m13:04:50.744760 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 1.56s]
[0m13:04:50.745461 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
```

There's nothing too special going on, we create our microbatch model for the first time selecting from the raw data filtering for only the day `2025-03-01`. 

The current state of `my_first_microbatch` looks like:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |

#### Day 2 (2025-03-02)

Day 2 rolls around, and let's update our raw data, since overnight, our data loading processess would have done it's job:

```sql
insert into db.raw.customers as (2, 'bob', '2025-03-02'::date);
```

And our daily dbt job kicks off:

<details>
<summary>Expand...</summary>

```sh
$ sudo date -u 0302000025
$ dbt build

[0m13:00:46.907592 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:46.908955 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:00:46.909858 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:00:46.910603 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:46.911215 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:46.914567 [debug] [Thread-1 (]: batch 2025-03-01 of sch.my_first_microbatch is being run sequentially
[0m13:00:46.915346 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:46.916234 [info ] [Thread-1 (]: Batch 1 of 2 START batch 2025-03-01 of sch.my_first_microbatch ....................... [RUN]
[0m13:00:46.917432 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:47.030780 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:47.032470 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:47.301256 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:47.302196 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250301
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-01 00:00:00+00:00' and loaded_at < '2025-03-02 00:00:00+00:00')
        );
[0m13:00:48.315712 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.013 seconds
[0m13:00:48.335437 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:48.336505 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250301
[0m13:00:48.612655 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.274 seconds
[0m13:00:48.622863 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:48.624032 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:00:48.839266 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.214 seconds
[0m13:00:48.860439 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:48.861468 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:00:49.139895 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.277 seconds
[0m13:00:49.167129 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:49.169276 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:49.169841 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-01 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-02 00:00:00+00:00')
    
    );
[0m13:00:49.528850 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.358 seconds
[0m13:00:49.530434 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:49.531650 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250301
    )
[0m13:00:50.158807 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.626 seconds
[0m13:00:50.180741 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250301
[0m13:00:50.192902 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:50.193523 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250301 cascade
[0m13:00:50.488227 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.294 seconds
[0m13:00:50.525988 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '164bcd3d-c8bd-443a-b93a-ff3ef0bab570', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x106a6f290>]}
[0m13:00:50.527015 [info ] [Thread-1 (]: Batch 1 of 2 OK created batch 2025-03-01 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 3.61s]
[0m13:00:50.528032 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:00:50.529169 [debug] [Thread-1 (]: batch 2025-03-02 of sch.my_first_microbatch is being run sequentially
[0m13:00:50.530166 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:50.531046 [info ] [Thread-1 (]: Batch 2 of 2 START batch 2025-03-02 of sch.my_first_microbatch ....................... [RUN]
[0m13:00:50.532377 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:50.537117 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:50.538387 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:50.543492 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:50.544505 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250302
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-02 00:00:00+00:00' and loaded_at < '2025-03-03 00:00:00+00:00')
        );
[0m13:00:51.285929 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.741 seconds
[0m13:00:51.289027 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:51.289697 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250302
[0m13:00:51.506213 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.216 seconds
[0m13:00:51.515324 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:51.516414 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:00:51.715159 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.198 seconds
[0m13:00:51.724321 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:51.725498 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:00:51.930838 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.204 seconds
[0m13:00:51.934713 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:51.936930 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:51.938130 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-02 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-03 00:00:00+00:00')
    
    );
[0m13:00:52.278254 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.339 seconds
[0m13:00:52.279788 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:52.280956 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250302
    )
[0m13:00:52.922634 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.640 seconds
[0m13:00:52.930729 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250302
[0m13:00:52.933164 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:52.934276 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250302 cascade
[0m13:00:53.201457 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.265 seconds
[0m13:00:53.206907 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '164bcd3d-c8bd-443a-b93a-ff3ef0bab570', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1070f5ad0>]}
[0m13:00:53.208685 [info ] [Thread-1 (]: Batch 2 of 2 OK created batch 2025-03-02 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 2.67s]
[0m13:00:53.210307 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:00:53.212014 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '164bcd3d-c8bd-443a-b93a-ff3ef0bab570', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x107d4d810>]}
[0m13:00:53.213501 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 6.30s]
```

</details>

State of `my_first_microbatch` table as of `2025-03-02`:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |


#### Day 3 (2025-03-03)

Day 3 rolls around, and let's update our raw data again as the overnight data load has been completed:

```sql
insert into db.raw.customers as (3, 'eve', '2025-03-03'::date);
```

And our daily dbt job kicks off:

<details>
<summary>Expand...</summary>

```sh
$ sudo date -u 0303000025
$ dbt build

[0m13:00:17.258175 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:00:17.258810 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:00:17.259331 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:17.260102 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:17.264469 [debug] [Thread-1 (]: batch 2025-03-02 of sch.my_first_microbatch is being run sequentially
[0m13:00:17.266021 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:17.266938 [info ] [Thread-1 (]: Batch 1 of 2 START batch 2025-03-02 of sch.my_first_microbatch ....................... [RUN]
[0m13:00:17.267779 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:17.280213 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:17.282014 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:17.357203 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:17.358018 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250302
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-02 00:00:00+00:00' and loaded_at < '2025-03-03 00:00:00+00:00')
        );
[0m13:00:18.425161 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.066 seconds
[0m13:00:18.484780 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:18.485434 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250302
[0m13:00:18.735595 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.249 seconds
[0m13:00:18.744775 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:18.746264 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:00:18.945809 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.198 seconds
[0m13:00:18.967336 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:18.968199 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:00:19.172416 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.203 seconds
[0m13:00:19.198826 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:19.200868 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:19.201589 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-02 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-03 00:00:00+00:00')
    
    );
[0m13:00:19.962895 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.760 seconds
[0m13:00:19.964411 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:19.965589 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250302
    )
[0m13:00:20.576986 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.610 seconds
[0m13:00:20.596714 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250302
[0m13:00:20.609360 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:20.610022 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250302 cascade
[0m13:00:20.885058 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.274 seconds
[0m13:00:20.928925 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'a3b38623-f301-4db9-b759-58ab9648be79', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x106cfaf50>]}
[0m13:00:20.930115 [info ] [Thread-1 (]: Batch 1 of 2 OK created batch 2025-03-02 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 3.66s]
[0m13:00:20.930887 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:00:20.931583 [debug] [Thread-1 (]: batch 2025-03-03 of sch.my_first_microbatch is being run sequentially
[0m13:00:20.932028 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:20.932642 [info ] [Thread-1 (]: Batch 2 of 2 START batch 2025-03-03 of sch.my_first_microbatch ....................... [RUN]
[0m13:00:20.933235 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:20.939230 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:20.940544 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:20.947950 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:20.948814 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250303
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-03 00:00:00+00:00' and loaded_at < '2025-03-04 00:00:00+00:00')
        );
[0m13:00:21.702787 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.753 seconds
[0m13:00:21.710879 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:21.712497 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250303
[0m13:00:21.975087 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.261 seconds
[0m13:00:21.983804 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:21.984966 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:00:22.179866 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.194 seconds
[0m13:00:22.188867 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:22.190021 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:00:22.392923 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.201 seconds
[0m13:00:22.397116 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:22.399340 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:22.400368 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-03 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-04 00:00:00+00:00')
    
    );
[0m13:00:22.825545 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.424 seconds
[0m13:00:22.827319 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:22.828414 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250303
    )
[0m13:00:23.545628 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.716 seconds
[0m13:00:23.553679 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250303
[0m13:00:23.555840 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:23.556841 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250303 cascade
[0m13:00:23.811423 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.254 seconds
[0m13:00:23.816591 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'a3b38623-f301-4db9-b759-58ab9648be79', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x106e03810>]}
[0m13:00:23.818779 [info ] [Thread-1 (]: Batch 2 of 2 OK created batch 2025-03-03 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 2.88s]
[0m13:00:23.820562 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:00:23.822486 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'a3b38623-f301-4db9-b759-58ab9648be79', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1057e8290>]}
[0m13:00:23.823931 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 6.56s]
```

</details>

State of `my_first_microbatch` table as of `2025-03-03`:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |

#### Batch lifecycle

The pattern (emitted SQL) for day 3 is basically the same to day 2 above with the SQL DDL/DML taking on the form of, for each batch:

1. Create a temporary table for a particular batch.
2. Delete from the target table any rows that match the particular batch.
3. Insert into the target table, selecting from (1).

Let's do a few more test scenarios to see if the batching behaviour determined above is consistent - what happens if our raw data had been continuously updated but our dbt job had failed to run for whatever reason.

#### Day 4 (2025-03-04)

```sql
insert into db.raw.customers values (4, 'carol', '2025-03-04'::date);
```

EL process worked and the raw source was updated but dbt did not run the model for whatever reason. 

State of `my_first_microbatch` table as of `2025-03-04`:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |


#### Day 5 (2025-03-05)

```sql
insert into db.raw.customers values (5, 'dave', '2025-03-05'::date);
```

EL process worked and the raw source was updated but dbt did not run the model for whatever reason. 

State of `my_first_microbatch` table as of `2025-03-05`:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |

#### Day 6 (2025-03-06)

```sql
insert into db.raw.customers values (6, 'fay', '2025-03-06'::date);
```

EL process worked and the raw source was updated but dbt did not run the model for whatever reason. 

State of `my_first_microbatch` table as of `2025-03-06`:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |

#### Day 7 (2025-03-07)

```sql
insert into db.raw.customers values (7, 'grace', '2025-03-07'::date);
```

After missing 3 days of dbt runs (day 4, 5 and 6), what happens on day 7 if dbt starts to run the model again:

<details>
<summary>Expand...</summary>

```sh
$ sudo date -u 0307000025
$ dbt build

[0m13:01:02.634610 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:01:02.635677 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:01:02.636467 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:01:02.637441 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:01:02.638371 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:01:02.643052 [debug] [Thread-1 (]: batch 2025-03-06 of sch.my_first_microbatch is being run sequentially
[0m13:01:02.643773 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:01:02.644413 [info ] [Thread-1 (]: Batch 1 of 2 START batch 2025-03-06 of sch.my_first_microbatch ....................... [RUN]
[0m13:01:02.645056 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:01:02.659799 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:01:02.660753 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:01:02.739111 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:02.739852 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250306
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-06 00:00:00+00:00' and loaded_at < '2025-03-07 00:00:00+00:00')
        );
[0m13:01:04.276815 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.536 seconds
[0m13:01:04.297407 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:04.299121 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250306
[0m13:01:04.626549 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.326 seconds
[0m13:01:04.635304 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:04.636577 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:01:04.895784 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.258 seconds
[0m13:01:04.914031 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:04.914873 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:01:05.160740 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.245 seconds
[0m13:01:05.189069 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:01:05.191086 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:05.191673 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-06 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-07 00:00:00+00:00')
    
    );
[0m13:01:05.648655 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.456 seconds
[0m13:01:05.650208 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:05.651380 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250306
    )
[0m13:01:06.460333 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.808 seconds
[0m13:01:06.478745 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250306
[0m13:01:06.491644 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:06.492307 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250306 cascade
[0m13:01:06.772320 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.279 seconds
[0m13:01:06.812835 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'cd7b7f5d-bf96-43a8-b37d-4f37bf9aab62', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b638f10>]}
[0m13:01:06.813997 [info ] [Thread-1 (]: Batch 1 of 2 OK created batch 2025-03-06 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 4.17s]
[0m13:01:06.814819 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:01:06.815655 [debug] [Thread-1 (]: batch 2025-03-07 of sch.my_first_microbatch is being run sequentially
[0m13:01:06.816216 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:01:06.816841 [info ] [Thread-1 (]: Batch 2 of 2 START batch 2025-03-07 of sch.my_first_microbatch ....................... [RUN]
[0m13:01:06.817508 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:01:06.822456 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:01:06.823639 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:01:06.832184 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:06.832886 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250307
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-07 00:00:00+00:00' and loaded_at < '2025-03-08 00:00:00+00:00')
        );
[0m13:01:07.859869 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.026 seconds
[0m13:01:07.866873 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:07.868242 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250307
[0m13:01:08.554594 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.685 seconds
[0m13:01:08.563049 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:08.564225 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:01:08.800870 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.236 seconds
[0m13:01:08.809342 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:08.810940 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:01:09.084462 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.272 seconds
[0m13:01:09.089927 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:01:09.093216 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:09.094458 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-07 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-08 00:00:00+00:00')
    
    );
[0m13:01:09.607186 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.511 seconds
[0m13:01:09.608699 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:09.609878 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250307
    )
[0m13:01:10.420087 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.809 seconds
[0m13:01:10.428214 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250307
[0m13:01:10.430323 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:10.431269 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250307 cascade
[0m13:01:10.844978 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.413 seconds
[0m13:01:10.850184 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'cd7b7f5d-bf96-43a8-b37d-4f37bf9aab62', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10c406710>]}
[0m13:01:10.852012 [info ] [Thread-1 (]: Batch 2 of 2 OK created batch 2025-03-07 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 4.03s]
[0m13:01:10.853664 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:01:10.855083 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'cd7b7f5d-bf96-43a8-b37d-4f37bf9aab62', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10c469550>]}
[0m13:01:10.856544 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 8.22s]
```

</details>

When dbt ran on day 7 (`2025-03-07`), after not running for 3 days (days 4, 5 and 6), the first batch that ran was for the date `2025-03-06` followed by the batch for the current date itself (`2025-03-07`). This means that our microbatch model necessarily missed out on some data - the state of the table as of `2025-03-07` after the dbt run is as follows:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |
| 6  | fay        | 2025-03-06 |
| 7  | grace      | 2025-03-07 |

Missing id's `4` and `5` from the source.

To do a backfill, because our daily job failed to run properly for those few days, we need to run an adhoc command with the `--event-time-start` and `--event-time-end` specified:

<details>
<summary>Expand...</summary>

```sh
$ date -u 
Fri Mar 7 00:09:10 UTC 2025

$ dbt build --event-time-start "2025-03-04" --event-time-end "2025-03-06"

[0m13:13:32.859250 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:13:32.860167 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:13:32.860790 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:13:32.861283 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:13:32.862051 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:13:32.866692 [debug] [Thread-1 (]: batch 2025-03-04 of sch.my_first_microbatch is being run sequentially
[0m13:13:32.868993 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:13:32.869772 [info ] [Thread-1 (]: Batch 1 of 2 START batch 2025-03-04 of sch.my_first_microbatch ....................... [RUN]
[0m13:13:32.870686 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:13:32.884704 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:13:32.885652 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:13:32.960124 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:32.960855 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250304
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-04 00:00:00+00:00' and loaded_at < '2025-03-05 00:00:00+00:00')
        );
[0m13:13:33.851797 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.890 seconds
[0m13:13:33.878124 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:33.879221 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250304
[0m13:13:34.192139 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.312 seconds
[0m13:13:34.200975 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:34.202292 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:13:34.487147 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.284 seconds
[0m13:13:34.506933 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:34.507822 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:13:34.729465 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.221 seconds
[0m13:13:34.755893 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:13:34.757850 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:34.758482 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-04 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-05 00:00:00+00:00')
    
    );
[0m13:13:35.288398 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.529 seconds
[0m13:13:35.289905 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:35.291075 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250304
    )
[0m13:13:35.902000 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.609 seconds
[0m13:13:35.921568 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250304
[0m13:13:35.935270 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:35.936144 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250304 cascade
[0m13:13:36.212971 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.276 seconds
[0m13:13:36.252151 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '24d7de52-75a7-4d98-aa1d-2be530bd6b56', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x104e97cd0>]}
[0m13:13:36.253396 [info ] [Thread-1 (]: Batch 1 of 2 OK created batch 2025-03-04 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 3.38s]
[0m13:13:36.254149 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:13:36.254895 [debug] [Thread-1 (]: batch 2025-03-05 of sch.my_first_microbatch is being run sequentially
[0m13:13:36.255441 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:13:36.256046 [info ] [Thread-1 (]: Batch 2 of 2 START batch 2025-03-05 of sch.my_first_microbatch ....................... [RUN]
[0m13:13:36.256684 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:13:36.261272 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:13:36.262402 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:13:36.270623 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:36.271276 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250305
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-05 00:00:00+00:00' and loaded_at < '2025-03-06 00:00:00+00:00')
        );
[0m13:13:37.132932 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.861 seconds
[0m13:13:37.140928 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:37.142217 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250305
[0m13:13:37.358902 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.215 seconds
[0m13:13:37.368070 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:37.369451 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:13:37.580317 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.210 seconds
[0m13:13:37.589507 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:37.590644 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:13:37.813625 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.222 seconds
[0m13:13:37.818830 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:13:37.822167 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:37.823340 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-05 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-06 00:00:00+00:00')
    
    );
[0m13:13:38.197188 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.373 seconds
[0m13:13:38.199620 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:38.200889 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250305
    )
[0m13:13:38.975638 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.773 seconds
[0m13:13:38.983341 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250305
[0m13:13:38.986135 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:13:38.987287 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250305 cascade
[0m13:13:39.254709 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.266 seconds
[0m13:13:39.259882 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '24d7de52-75a7-4d98-aa1d-2be530bd6b56', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x106f4dd90>]}
[0m13:13:39.261734 [info ] [Thread-1 (]: Batch 2 of 2 OK created batch 2025-03-05 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 3.00s]
[0m13:13:39.263402 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:13:39.265017 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '24d7de52-75a7-4d98-aa1d-2be530bd6b56', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1057f2310>]}
[0m13:13:39.266805 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 6.40s]

```

</details>

1. While we're running the adhoc run still on day 7 (as the output of `date -u` show), this is not a necessary condition for the data to be updated properly.
2. The `--event-time-end` is the ending boundary (`< event_time_end`) - meaning, if we did something like `--event-time-end "2025-03-05"` then we would not actually run a batch for that day/date itself (i.e. `2025-03-05`) when we actually would want to do so.

We processed 2 batches for dates `2025-03-04` and `2025-03-05` which then brings our microbatch model back to speed:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |
| 6  | fay        | 2025-03-06 |
| 7  | grace      | 2025-03-07 |
| 4  | carol      | 2025-03-04 |
| 5  | dave       | 2025-03-05 |

> This table is displayed per the order they were inserted in to illustrate the order of operations instead of ordering them by `id` or `updated_at`.

Here we determine that the default configuration is for dbt to run 1 batch behind the current batch just as the default `lookback` is configured to do (https://docs.getdbt.com/reference/resource-configs/lookback) - it's also possible to configure an extended lookback - for example, you know your models may not always run properly and so you want to always process the last 3 days or something of that nature.

Let's test a different scenario - let's have our microbatch model run everyday as expected, but our raw source data not be updated everyday - basically the reverse scenario.

#### Day 8 (2025-03-08)

Raw data did not update but dbt did run:

<details>
<summary>Expand...</summary>

```sh
$ sudo date -u 0308000025
$ dbt build

[0m13:00:10.204820 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:00:10.205888 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:00:10.206755 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:10.207394 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:10.210724 [debug] [Thread-1 (]: batch 2025-03-07 of sch.my_first_microbatch is being run sequentially
[0m13:00:10.211502 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:10.212801 [info ] [Thread-1 (]: Batch 1 of 2 START batch 2025-03-07 of sch.my_first_microbatch ....................... [RUN]
[0m13:00:10.215522 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:10.228921 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:10.229895 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:10.308040 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:10.308791 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250307
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-07 00:00:00+00:00' and loaded_at < '2025-03-08 00:00:00+00:00')
        );
[0m13:00:11.841373 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.532 seconds
[0m13:00:11.861730 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:11.862610 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250307
[0m13:00:12.082236 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.219 seconds
[0m13:00:12.090829 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:12.092096 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:00:12.313744 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.220 seconds
[0m13:00:12.332247 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:12.333673 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:00:12.539674 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.205 seconds
[0m13:00:12.565993 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:12.567958 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:12.568766 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-07 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-08 00:00:00+00:00')
    
    );
[0m13:00:13.168617 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.599 seconds
[0m13:00:13.170114 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:13.171293 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250307
    )
[0m13:00:13.896240 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.724 seconds
[0m13:00:13.916084 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250307
[0m13:00:13.927094 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:13.927801 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250307 cascade
[0m13:00:14.177326 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.249 seconds
[0m13:00:14.202655 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '92286cf2-b258-428a-aacd-8d8b57086b26', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10e2f6910>]}
[0m13:00:14.203599 [info ] [Thread-1 (]: Batch 1 of 2 OK created batch 2025-03-07 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 3.99s]
[0m13:00:14.204335 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:00:14.205079 [debug] [Thread-1 (]: batch 2025-03-08 of sch.my_first_microbatch is being run sequentially
[0m13:00:14.205618 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:14.206368 [info ] [Thread-1 (]: Batch 2 of 2 START batch 2025-03-08 of sch.my_first_microbatch ....................... [RUN]
[0m13:00:14.207122 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:14.211304 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:14.212420 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:14.220260 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:14.220938 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250308
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-08 00:00:00+00:00' and loaded_at < '2025-03-09 00:00:00+00:00')
        );
[0m13:00:15.010618 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.789 seconds
[0m13:00:15.015733 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:15.016936 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250308
[0m13:00:15.241091 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.223 seconds
[0m13:00:15.249458 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:15.250741 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:00:15.453688 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.202 seconds
[0m13:00:15.462559 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:15.463770 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:00:15.817681 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.353 seconds
[0m13:00:15.822735 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:15.826088 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:15.827377 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-08 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-09 00:00:00+00:00')
    
    );
[0m13:00:16.327116 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.498 seconds
[0m13:00:16.328667 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:16.329810 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250308
    )
[0m13:00:16.685585 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.354 seconds
[0m13:00:16.693811 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250308
[0m13:00:16.695987 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:16.696976 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250308 cascade
[0m13:00:17.006195 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.308 seconds
[0m13:00:17.011657 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '92286cf2-b258-428a-aacd-8d8b57086b26', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1088c6310>]}
[0m13:00:17.013763 [info ] [Thread-1 (]: Batch 2 of 2 OK created batch 2025-03-08 of sch.my_first_microbatch .................. [[32mSUCCESS 0[0m in 2.80s]
[0m13:00:17.015316 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:00:17.016595 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '92286cf2-b258-428a-aacd-8d8b57086b26', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10e335390>]}
[0m13:00:17.017817 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 6.81s]
```

</details>

The state of our microbatch model on `2025-03-08`:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |
| 6  | fay        | 2025-03-06 |
| 7  | grace      | 2025-03-07 |
| 4  | carol      | 2025-03-04 |
| 5  | dave       | 2025-03-05 |


#### Day 9 (2025-03-09)

Raw data did not update but dbt did run.

<details>
<summary>Expand...</summary>

```sh
$ sudo date -u 0309000025
$ dbt build

0m13:01:25.035170 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:01:25.035885 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:01:25.036435 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:01:25.036932 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:01:25.040802 [debug] [Thread-1 (]: batch 2025-03-08 of sch.my_first_microbatch is being run sequentially
[0m13:01:25.042201 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:01:25.044400 [info ] [Thread-1 (]: Batch 1 of 2 START batch 2025-03-08 of sch.my_first_microbatch ....................... [RUN]
[0m13:01:25.045326 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:01:25.058300 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:01:25.059479 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:01:25.135626 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:25.136422 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250308
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-08 00:00:00+00:00' and loaded_at < '2025-03-09 00:00:00+00:00')
        );
[0m13:01:26.207881 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.071 seconds
[0m13:01:26.228316 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:26.229737 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250308
[0m13:01:26.474645 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.244 seconds
[0m13:01:26.483535 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:26.484706 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:01:26.708140 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.222 seconds
[0m13:01:26.725888 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:26.727478 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:01:26.999229 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.270 seconds
[0m13:01:27.025015 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:01:27.026992 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:27.027829 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-08 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-09 00:00:00+00:00')
    
    );
[0m13:01:27.599057 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.569 seconds
[0m13:01:27.600595 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:27.602682 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250308
    )
[0m13:01:28.170406 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.565 seconds
[0m13:01:28.190788 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250308
[0m13:01:28.203454 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:28.204394 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250308 cascade
[0m13:01:28.571995 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.367 seconds
[0m13:01:28.612437 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '9f191268-7f23-426e-9cdc-b082ee49e103', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10e04ded0>]}
[0m13:01:28.613775 [info ] [Thread-1 (]: Batch 1 of 2 OK created batch 2025-03-08 of sch.my_first_microbatch .................. [[32mSUCCESS 0[0m in 3.57s]
[0m13:01:28.614760 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:01:28.615918 [debug] [Thread-1 (]: batch 2025-03-09 of sch.my_first_microbatch is being run sequentially
[0m13:01:28.616979 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:01:28.618022 [info ] [Thread-1 (]: Batch 2 of 2 START batch 2025-03-09 of sch.my_first_microbatch ....................... [RUN]
[0m13:01:28.618847 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:01:28.622814 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:01:28.623858 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:01:28.632490 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:28.634421 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250309
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-09 00:00:00+00:00' and loaded_at < '2025-03-10 00:00:00+00:00')
        );
[0m13:01:29.242427 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.607 seconds
[0m13:01:29.249607 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:29.250909 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250309
[0m13:01:29.487463 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.235 seconds
[0m13:01:29.492708 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:29.493540 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:01:29.738360 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.244 seconds
[0m13:01:29.744874 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:29.745961 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:01:29.967964 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.221 seconds
[0m13:01:29.970931 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:01:29.972714 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:29.973639 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-09 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-10 00:00:00+00:00')
    
    );
[0m13:01:30.346666 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.372 seconds
[0m13:01:30.348311 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:30.349706 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250309
    )
[0m13:01:30.707468 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.356 seconds
[0m13:01:30.716091 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250309
[0m13:01:30.718359 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:01:30.719398 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250309 cascade
[0m13:01:30.991683 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.271 seconds
[0m13:01:30.997118 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '9f191268-7f23-426e-9cdc-b082ee49e103', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10e0e2490>]}
[0m13:01:30.998765 [info ] [Thread-1 (]: Batch 2 of 2 OK created batch 2025-03-09 of sch.my_first_microbatch .................. [[32mSUCCESS 0[0m in 2.38s]
[0m13:01:31.000621 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:01:31.002230 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '9f191268-7f23-426e-9cdc-b082ee49e103', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x108f89a10>]}
[0m13:01:31.004304 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 5.97s]

```

</details>

The state of our microbatch model on `2025-03-09`:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |
| 6  | fay        | 2025-03-06 |
| 7  | grace      | 2025-03-07 |
| 4  | carol      | 2025-03-04 |
| 5  | dave       | 2025-03-05 |


#### Day 10 (2025-03-10)

Raw data did not update but dbt did run. 

<details>
<summary>Expand...</summary>

```sh
$ sudo date -u 0310000025
$ dbt build

[0m13:00:14.221334 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:00:14.221967 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:00:14.222508 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:14.223039 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:14.226748 [debug] [Thread-1 (]: batch 2025-03-09 of sch.my_first_microbatch is being run sequentially
[0m13:00:14.228301 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:14.229182 [info ] [Thread-1 (]: Batch 1 of 2 START batch 2025-03-09 of sch.my_first_microbatch ....................... [RUN]
[0m13:00:14.229842 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:14.242869 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:14.244457 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:14.321810 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:14.322580 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250309
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-09 00:00:00+00:00' and loaded_at < '2025-03-10 00:00:00+00:00')
        );
[0m13:00:15.485186 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.162 seconds
[0m13:00:15.505379 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:15.506292 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250309
[0m13:00:15.790735 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.283 seconds
[0m13:00:15.799587 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:15.800867 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:00:16.096976 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.295 seconds
[0m13:00:16.115488 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:16.117093 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:00:16.404950 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.287 seconds
[0m13:00:16.429554 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:16.431680 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:16.433273 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-09 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-10 00:00:00+00:00')
    
    );
[0m13:00:16.789522 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.355 seconds
[0m13:00:16.791182 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:16.792636 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250309
    )
[0m13:00:17.186005 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.392 seconds
[0m13:00:17.206948 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250309
[0m13:00:17.218478 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:17.219209 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250309 cascade
[0m13:00:17.520476 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.300 seconds
[0m13:00:17.559791 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'd1b2fd69-812a-499e-a65d-7cb075fb8355', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x110f1fcd0>]}
[0m13:00:17.560707 [info ] [Thread-1 (]: Batch 1 of 2 OK created batch 2025-03-09 of sch.my_first_microbatch .................. [[32mSUCCESS 0[0m in 3.33s]
[0m13:00:17.561416 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:00:17.562306 [debug] [Thread-1 (]: batch 2025-03-10 of sch.my_first_microbatch is being run sequentially
[0m13:00:17.562794 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:00:17.563292 [info ] [Thread-1 (]: Batch 2 of 2 START batch 2025-03-10 of sch.my_first_microbatch ....................... [RUN]
[0m13:00:17.563823 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:00:17.568270 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:17.570569 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:00:17.578946 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:17.579766 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250310
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-10 00:00:00+00:00' and loaded_at < '2025-03-11 00:00:00+00:00')
        );
[0m13:00:18.272465 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.692 seconds
[0m13:00:18.279571 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:18.280845 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250310
[0m13:00:18.592801 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.310 seconds
[0m13:00:18.601449 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:18.602986 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:00:18.938161 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.334 seconds
[0m13:00:18.943148 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:18.944600 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:00:19.419981 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.454 seconds
[0m13:00:19.425137 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:00:19.428379 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:19.429504 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-10 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-11 00:00:00+00:00')
    
    );
[0m13:00:19.762437 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.332 seconds
[0m13:00:19.764550 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:19.766268 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250310
    )
[0m13:00:20.138266 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.370 seconds
[0m13:00:20.147214 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250310
[0m13:00:20.149112 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:00:20.151546 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250310 cascade
[0m13:00:20.468446 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.315 seconds
[0m13:00:20.473565 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'd1b2fd69-812a-499e-a65d-7cb075fb8355', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x116179550>]}
[0m13:00:20.475412 [info ] [Thread-1 (]: Batch 2 of 2 OK created batch 2025-03-10 of sch.my_first_microbatch .................. [[32mSUCCESS 0[0m in 2.91s]
[0m13:00:20.477051 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:00:20.478440 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'd1b2fd69-812a-499e-a65d-7cb075fb8355', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x116125910>]}
[0m13:00:20.479920 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 6.26s]

```

</details>


The state of our microbatch model on `2025-03-10`:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |
| 6  | fay        | 2025-03-06 |
| 7  | grace      | 2025-03-07 |
| 4  | carol      | 2025-03-04 |
| 5  | dave       | 2025-03-05 |

The behaviour is basically identical to what we had before... we created a temporary table for the batch (e.g. `2025-03-10`), but there was no data in the raw source table for that particular batch - therefore, when the insert happened for that batch, there was nothing to insert (we see `SUCCESS 0` in the logs for the insert query).

#### Day 11 (2025-03-11)

On day 11, our raw data loader caught up and loaded all the missing data:

```sql
insert into db.raw.customers values (8, 'heidi', '2025-03-08'::date);
insert into db.raw.customers values (9, 'ivan', '2025-03-09'::date);
insert into db.raw.customers values (10, 'judy', '2025-03-10'::date);
insert into db.raw.customers values (11, 'mike', '2025-03-11'::date);
```

And our dbt run:

<details>
<summary>Expand...</summary>

```sh
$ sudo date -u 0311000025
$ dbt build

0m13:02:52.228261 [info ] [Thread-1 (]: 1 of 1 START sql microbatch model sch.my_first_microbatch ...................... [RUN]
[0m13:02:52.228953 [debug] [Thread-1 (]: Re-using an available connection from the pool (formerly list_db_sch, now model.my_dbt_project.my_first_microbatch)
[0m13:02:52.229507 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:02:52.230011 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:02:52.233270 [debug] [Thread-1 (]: batch 2025-03-10 of sch.my_first_microbatch is being run sequentially
[0m13:02:52.233906 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:02:52.234559 [info ] [Thread-1 (]: Batch 1 of 2 START batch 2025-03-10 of sch.my_first_microbatch ....................... [RUN]
[0m13:02:52.235236 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:02:52.249189 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:02:52.250309 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:02:52.326144 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:52.326918 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250310
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-10 00:00:00+00:00' and loaded_at < '2025-03-11 00:00:00+00:00')
        );
[0m13:02:53.685192 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 1.357 seconds
[0m13:02:53.706496 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:53.707650 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250310
[0m13:02:53.923666 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.215 seconds
[0m13:02:53.932426 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:53.933678 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:02:54.212719 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.278 seconds
[0m13:02:54.231148 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:54.232407 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:02:54.451935 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.218 seconds
[0m13:02:54.478492 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:02:54.480835 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:54.481565 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-10 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-11 00:00:00+00:00')
    
    );
[0m13:02:55.083384 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.601 seconds
[0m13:02:55.084850 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:55.086339 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250310
    )
[0m13:02:55.936700 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.849 seconds
[0m13:02:55.958508 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250310
[0m13:02:55.969628 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:55.970725 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250310 cascade
[0m13:02:56.208953 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.237 seconds
[0m13:02:56.238922 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '2a3d4974-98f6-435f-a5cd-6c0e967502f1', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b2aa990>]}
[0m13:02:56.239878 [info ] [Thread-1 (]: Batch 1 of 2 OK created batch 2025-03-10 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 4.00s]
[0m13:02:56.240662 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:02:56.241529 [debug] [Thread-1 (]: batch 2025-03-11 of sch.my_first_microbatch is being run sequentially
[0m13:02:56.242158 [debug] [Thread-1 (]: Began running node model.my_dbt_project.my_first_microbatch
[0m13:02:56.243020 [info ] [Thread-1 (]: Batch 2 of 2 START batch 2025-03-11 of sch.my_first_microbatch ....................... [RUN]
[0m13:02:56.243663 [debug] [Thread-1 (]: Began compiling node model.my_dbt_project.my_first_microbatch
[0m13:02:56.247808 [debug] [Thread-1 (]: Writing injected SQL for node "model.my_dbt_project.my_first_microbatch"
[0m13:02:56.248780 [debug] [Thread-1 (]: Began executing node model.my_dbt_project.my_first_microbatch
[0m13:02:56.255870 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:56.256573 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250311
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-11 00:00:00+00:00' and loaded_at < '2025-03-12 00:00:00+00:00')
        );
[0m13:02:57.251548 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.994 seconds
[0m13:02:57.258871 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:57.260228 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch__dbt_tmp_20250311
[0m13:02:57.658992 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.398 seconds
[0m13:02:57.664777 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:57.666172 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table db.sch.my_first_microbatch
[0m13:02:57.936694 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.269 seconds
[0m13:02:57.945035 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:57.946273 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
describe table "DB"."SCH"."MY_FIRST_MICROBATCH"
[0m13:02:58.167511 [debug] [Thread-1 (]: SQL status: SUCCESS 3 in 0.220 seconds
[0m13:02:58.170369 [debug] [Thread-1 (]: Writing runtime sql for node "model.my_dbt_project.my_first_microbatch"
[0m13:02:58.172184 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:58.172796 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-11 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-12 00:00:00+00:00')
    
    );
[0m13:02:58.521479 [debug] [Thread-1 (]: SQL status: SUCCESS 0 in 0.347 seconds
[0m13:02:58.523505 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:58.524759 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250311
    )
[0m13:02:59.299570 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.773 seconds
[0m13:02:59.307279 [debug] [Thread-1 (]: Applying DROP to: db.sch.my_first_microbatch__dbt_tmp_20250311
[0m13:02:59.310128 [debug] [Thread-1 (]: Using snowflake connection "model.my_dbt_project.my_first_microbatch"
[0m13:02:59.311517 [debug] [Thread-1 (]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
drop table if exists db.sch.my_first_microbatch__dbt_tmp_20250311 cascade
[0m13:02:59.590369 [debug] [Thread-1 (]: SQL status: SUCCESS 1 in 0.277 seconds
[0m13:02:59.595587 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '2a3d4974-98f6-435f-a5cd-6c0e967502f1', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b583510>]}
[0m13:02:59.598081 [info ] [Thread-1 (]: Batch 2 of 2 OK created batch 2025-03-11 of sch.my_first_microbatch .................. [[32mSUCCESS 1[0m in 3.35s]
[0m13:02:59.601205 [debug] [Thread-1 (]: Finished running node model.my_dbt_project.my_first_microbatch
[0m13:02:59.603347 [debug] [Thread-1 (]: Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '2a3d4974-98f6-435f-a5cd-6c0e967502f1', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b51a150>]}
[0m13:02:59.605604 [info ] [Thread-1 (]: 1 of 1 OK created sql microbatch model sch.my_first_microbatch ................. [[32mSUCCESS[0m in 7.37s]

```

</details>

As we determined above, dbt will start the previous batch from the current date (`2025-03-11`) - i.e. `2025-03-10`, therefore the model will, just like before, be missing some rows of data:

| id | first_name | updated_at |
|----|------------|------------|
| 1  | alice      | 2025-03-01 |
| 2  | bob        | 2025-03-02 |
| 3  | eve        | 2025-03-03 |
| 6  | fay        | 2025-03-06 |
| 4  | carol      | 2025-03-04 |
| 5  | dave       | 2025-03-05 |
| 7  | grace      | 2025-03-07 |
| 10 | judy       | 2025-03-10 |
| 11 | mike       | 2025-03-11 |

Missing id's `8` and `9` which we can resolve via adhoc runs with the `--event-time-start` / `--event-time-end` flags just like we did above.

Keep in mind that if we look at the debug logs for days 10 and 11 - we would see exactly identical SQL statments, from the creation of the temp, to the deletion and insertion into the target table:

```sh
[13:00:17.579766 [debug] [Thread-1]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250310
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-10 00:00:00+00:00' and loaded_at < '2025-03-11 00:00:00+00:00')
        );
[13:00:18.272465 [debug] [Thread-1]: SQL status: SUCCESS 1 in 0.692 seconds

[13:00:19.429504 [debug] [Thread-1]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-10 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-11 00:00:00+00:00')
    
    );
[13:00:19.762437 [debug] [Thread-1]: SQL status: SUCCESS 0 in 0.332 seconds

[13:00:19.766268 [debug] [Thread-1]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250310
    )
[13:00:20.138266 [debug] [Thread-1]: SQL status: SUCCESS 0 in 0.370 seconds
```

```sh
[13:02:52.326918 [debug] [Thread-1]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
create or replace temporary table db.sch.my_first_microbatch__dbt_tmp_20250310
         as
        (

select id, first_name, loaded_at as updated_at from (select * from db.raw.customers where loaded_at >= '2025-03-10 00:00:00+00:00' and loaded_at < '2025-03-11 00:00:00+00:00')
        );
[13:02:53.685192 [debug] [Thread-1]: SQL status: SUCCESS 1 in 1.357 seconds

[13:02:54.481565 [debug] [Thread-1]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
delete from db.sch.my_first_microbatch DBT_INTERNAL_TARGET
    where (
    DBT_INTERNAL_TARGET.updated_at >= to_timestamp_tz('2025-03-10 00:00:00+00:00')
    and DBT_INTERNAL_TARGET.updated_at < to_timestamp_tz('2025-03-11 00:00:00+00:00')
    
    );
[13:02:55.083384 [debug] [Thread-1]: SQL status: SUCCESS 0 in 0.601 seconds

[13:02:55.086339 [debug] [Thread-1]: On model.my_dbt_project.my_first_microbatch: /* {"app": "dbt", "dbt_version": "1.9.3", "profile_name": "all", "target_name": "default", "node_id": "model.my_dbt_project.my_first_microbatch"} */
insert into db.sch.my_first_microbatch ("ID", "FIRST_NAME", "UPDATED_AT")
    (
        select "ID", "FIRST_NAME", "UPDATED_AT"
        from db.sch.my_first_microbatch__dbt_tmp_20250310
    )
[13:02:55.936700 [debug] [Thread-1]: SQL status: SUCCESS 1 in 0.849 seconds
```

With the key difference being `SUCCESS 0` vs `SUCCESS 1` on the `insert` on day 10 vs day 11 - which makes sense because the temp table `my_first_microbatch__dbt_tmp_20250310` created on day 10 had 0 rows of data but on day 11, it had 1 row of data.

----

### Dynamically limiting the `begin` config depending on the environment / target

Sometimes, we may want to dynamically change the `begin` config - say for CI jobs since we don't want the microbatch model to be creating in our PR schema starting from the very beginning which can take a long time or be costly.

```sql
-- macros/limit_begin.sql
{% macro limit_begin(initial_date, days_prior=2) %}
    /*{# 
    Due to partial parsing, the value returned may be stuck on a previous parse. 
    Therefore we need to use an env var that changes every run - such as the dbt Cloud Run ID.
    https://github.com/dbt-labs/dbt-core/issues/4364

    If this is being used outside of a dbt Cloud run, then we may need to disable partial
    parsing to get it to reevaluate via adding the `--no-partial-parse` flag to the invocation.
    #}*/
    {% set run_id = env_var("DBT_CLOUD_RUN_ID", 'not-a-dbt-cloud-run') %}
    {% if target.name == 'prod' %}
        {{ return(initial_date) }}
    {% else %}
        {% set alternate_date = (modules.datetime.datetime.today() - modules.datetime.timedelta(days=days_prior)).strftime('%Y-%m-%d') %}
        {{ return(alternate_date) }}
    {% endif %}
{% endmacro %}

-- models/events.sql
{{ 
    config(
        materialized = 'incremental',
        incremental_strategy = 'microbatch',
        event_time = 'updated_at',
        begin = limit_begin('2025-04-01', 2),
        batch_size = 'day',
        concurrent_batches = false
    ) 
}}

select 1 as id, '2025-04-25'::date as updated_at
```

What will happen is that when the microbatch model runs and the `target.name` isn't 'prod' - the limit_being macro will return a date string which is 2 days prior to today...

```sh
$ date
Wed Apr 30 12:49:45 NZST 2025

$ dbt run --target ci
...
00:50:59  Concurrency: 1 threads (target='ci')
00:50:59  
00:51:02  1 of 1 START sql microbatch model dbt_cloud_pr_123.events ...................... [RUN]
00:51:02  Batch 1 of 3 START batch 2025-04-28 of dbt_cloud_pr_123.events ....................... [RUN]
00:51:03  Batch 1 of 3 OK created batch 2025-04-28 of dbt_cloud_pr_123.events .................. [SUCCESS 1 in 1.58s]
00:51:03  Batch 2 of 3 START batch 2025-04-29 of dbt_cloud_pr_123.events ....................... [RUN]
00:51:07  Batch 2 of 3 OK created batch 2025-04-29 of dbt_cloud_pr_123.events .................. [SUCCESS 1 in 3.52s]
00:51:07  Batch 3 of 3 START batch 2025-04-30 of dbt_cloud_pr_123.events ....................... [RUN]
00:51:10  Batch 3 of 3 OK created batch 2025-04-30 of dbt_cloud_pr_123.events .................. [SUCCESS 1 in 3.64s]
00:51:10  1 of 1 OK created sql microbatch model dbt_cloud_pr_123.events ................. [SUCCESS in 8.75s]
```

And if it's in prod, the limit_macro simply returns `2025-04-01` and that's what the `begin` config will be set to:

```sh
$ dbt run --target prod
...
00:52:43  Concurrency: 1 threads (target='prod')
00:52:43  
00:52:46  1 of 1 START sql microbatch model prod.events .................................. [RUN]
00:52:46  Batch 1 of 30 START batch 2025-04-01 of prod.events .................................. [RUN]
00:52:47  Batch 1 of 30 OK created batch 2025-04-01 of prod.events ............................. [SUCCESS 1 in 1.26s]
00:52:47  Batch 2 of 30 START batch 2025-04-02 of prod.events .................................. [RUN]
00:52:50  Batch 2 of 30 OK created batch 2025-04-02 of prod.events ............................. [SUCCESS 1 in 3.15s]
00:52:50  Batch 3 of 30 START batch 2025-04-03 of prod.events .................................. [RUN]
00:52:53  Batch 3 of 30 OK created batch 2025-04-03 of prod.events ............................. [SUCCESS 1 in 3.44s]
<TRUNCATED>
00:54:11  Batch 28 of 30 START batch 2025-04-28 of prod.events ................................. [RUN]
00:54:14  Batch 28 of 30 OK created batch 2025-04-28 of prod.events ............................ [SUCCESS 1 in 3.08s]
00:54:14  Batch 29 of 30 START batch 2025-04-29 of prod.events ................................. [RUN]
00:54:17  Batch 29 of 30 OK created batch 2025-04-29 of prod.events ............................ [SUCCESS 1 in 3.15s]
00:54:17  Batch 30 of 30 START batch 2025-04-30 of prod.events ................................. [RUN]
00:54:20  Batch 30 of 30 OK created batch 2025-04-30 of prod.events ............................ [SUCCESS 1 in 3.03s]
00:54:20  1 of 1 OK created sql microbatch model prod.events ............................. [SUCCESS in 94.95s]
```