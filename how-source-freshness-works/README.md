---
---

## How source freshness works

Video: https://youtu.be/-RFB1aA6gG4

Useful links:
* https://docs.getdbt.com/docs/build/sources#snapshotting-source-data-freshness
* https://docs.getdbt.com/docs/deploy/source-freshness
* https://docs.getdbt.com/reference/resource-properties/freshness

### How it works

At a high level, source freshness check are pretty straightforward:

1. (`max_loaded_at`) Retrieve the timestamp of the last time the data was updated.
    * [This will be converted into UTC if the timestamp is not in UTC](https://github.com/dbt-labs/dbt-adapters/blob/0739850873de0af14bc5506accb597256f0cdad1/dbt/adapters/base/impl.py#L1310).
    * The column that will be used is specified by the `loaded_at_field` property.
    * If the `loaded_at_field` property is not specified - then [use a metadata field on the table if possible](https://github.com/dbt-labs/dbt-core/blob/c215697a02d7c452c64d4d66f7ba77cf6f2181b6/core/dbt/task/freshness.py#L124) (e.g. in Snowflake - there is a `LAST_ALTERED` column in the information schema).
2. (`snapshotted_at`) Retrieve the current timestamp in UTC.
3. (`max_loaded_at_time_ago_in_s`) Calculate the difference between (1) and (2) (`snapshotted_at - max_loaded_at`). 
4. Check if `max_loaded_at_time_ago_in_s` exceeds the `warn_after` / `error_after` properties or not and pass / warn / error accordingly.

### Example

> The following example uses Snowflake.

```sql
show parameters like 'TIMEZONE' in account;
-- 'America/Los_Angeles'
```

We can see that on the account level - the timezone is 'America/Los_Angeles' so any timestamps that have TZ information created will be in `America/Los_Angeles` which at the time of this writing is currently `UTC-7`. 

Let's create a few different source tables we're going to do our source freshness checks on:

```sql
-- current_timestamp() will be in account timezone.
create or replace table src_la as (
select 1 id,
       current_timestamp() as updated_at,
       convert_timezone('UTC', updated_at) as updated_at_utc,
       typeof(updated_at::variant) updated_at_type
);

-- 'Pacific/Auckland' is 'UTC+12' as of this writing.
create or replace table src_akl as (
select 1 id,
       convert_timezone('Pacific/Auckland', current_timestamp()) as updated_at,
       convert_timezone('UTC', updated_at) as updated_at_utc,
       typeof(updated_at::variant) updated_at_type
);

-- sysdate() always returns utc.
create or replace table src_utc as (
select 1 id,
       sysdate() as updated_at,
       convert_timezone('UTC', 'UTC', updated_at) as updated_at_utc,
       typeof(updated_at::variant) updated_at_type
);
```

```sql
select * from src_la;
```
```
+----+-------------------------------+-------------------------------+-----------------+
| ID | UPDATED_AT                    | UPDATED_AT_UTC                | UPDATED_AT_TYPE |
+====+===============================+===============================+=================+
| 1  | 2024-06-30 20:21:21.029 -0700 | 2024-07-01 03:21:21.029 +0000 | TIMESTAMP_LTZ   |
+----+-------------------------------+-------------------------------+-----------------+
```

```sql
select * from src_akl;
```
```
+----+-------------------------------+-------------------------------+-----------------+
| ID | UPDATED_AT                    | UPDATED_AT_UTC                | UPDATED_AT_TYPE |
+====+===============================+===============================+=================+
| 1  | 2024-07-01 15:21:22.570 +1200 | 2024-07-01 03:21:22.570 +0000 | TIMESTAMP_TZ    |
+----+-------------------------------+-------------------------------+-----------------+
```

```sql
select * from src_utc;
```
```
+----+-------------------------------+-------------------------------+-----------------+
| ID | UPDATED_AT                    | UPDATED_AT_UTC                | UPDATED_AT_TYPE |
+====+===============================+===============================+=================+
| 1  | 2024-07-01 03:21:24.298       | 2024-07-01 03:21:24.298       | TIMESTAMP_NTZ   |
+----+-------------------------------+-------------------------------+-----------------+
```

Add our sources to dbt:

```yaml
# models/sources.yml
version: 2
sources:
  - name: dbt_jyeo
    freshness:
      error_after: {count: 1, period: hour}
    loaded_at_field: update_at
    tables:
      - name: src_la
      - name: src_akl
      - name: src_utc
```

And do a `source freshness` check:

```sh
$ dbt --debug source freshness
3:26:21  Began running node source.my_dbt_project.dbt_jyeo.src_akl
03:26:21  1 of 3 START freshness of dbt_jyeo.src_akl ..................................... [RUN]
03:26:21  Acquiring new snowflake connection 'source.my_dbt_project.dbt_jyeo.src_akl'
03:26:21  Began compiling node source.my_dbt_project.dbt_jyeo.src_akl
03:26:21  Began executing node source.my_dbt_project.dbt_jyeo.src_akl
03:26:21  Using snowflake connection "source.my_dbt_project.dbt_jyeo.src_akl"
03:26:21  On source.my_dbt_project.dbt_jyeo.src_akl: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "node_id": "source.my_dbt_project.dbt_jyeo.src_akl"} */
select
      max(updated_at) as max_loaded_at,
      convert_timezone('UTC', current_timestamp()) as snapshotted_at
    from development_jyeo.dbt_jyeo.src_akl
03:26:21  Opening a new connection, currently in state init
03:26:23  SQL status: SUCCESS 1 in 2.0 seconds
03:26:23  On source.my_dbt_project.dbt_jyeo.src_akl: Close
03:26:24  1 of 3 PASS freshness of dbt_jyeo.src_akl ...................................... [PASS in 2.54s]
03:26:24  Finished running node source.my_dbt_project.dbt_jyeo.src_akl
03:26:24  Began running node source.my_dbt_project.dbt_jyeo.src_la
03:26:24  2 of 3 START freshness of dbt_jyeo.src_la ...................................... [RUN]
03:26:24  Re-using an available connection from the pool (formerly source.my_dbt_project.dbt_jyeo.src_akl, now source.my_dbt_project.dbt_jyeo.src_la)
03:26:24  Began compiling node source.my_dbt_project.dbt_jyeo.src_la
03:26:24  Began executing node source.my_dbt_project.dbt_jyeo.src_la
03:26:24  Using snowflake connection "source.my_dbt_project.dbt_jyeo.src_la"
03:26:24  On source.my_dbt_project.dbt_jyeo.src_la: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "node_id": "source.my_dbt_project.dbt_jyeo.src_la"} */
select
      max(updated_at) as max_loaded_at,
      convert_timezone('UTC', current_timestamp()) as snapshotted_at
    from development_jyeo.dbt_jyeo.src_la
03:26:24  Opening a new connection, currently in state closed
03:26:25  SQL status: SUCCESS 1 in 2.0 seconds
03:26:25  On source.my_dbt_project.dbt_jyeo.src_la: Close
03:26:26  2 of 3 PASS freshness of dbt_jyeo.src_la ....................................... [PASS in 2.19s]
03:26:26  Finished running node source.my_dbt_project.dbt_jyeo.src_la
03:26:26  Began running node source.my_dbt_project.dbt_jyeo.src_utc
03:26:26  3 of 3 START freshness of dbt_jyeo.src_utc ..................................... [RUN]
03:26:26  Re-using an available connection from the pool (formerly source.my_dbt_project.dbt_jyeo.src_la, now source.my_dbt_project.dbt_jyeo.src_utc)
03:26:26  Began compiling node source.my_dbt_project.dbt_jyeo.src_utc
03:26:26  Began executing node source.my_dbt_project.dbt_jyeo.src_utc
03:26:26  Using snowflake connection "source.my_dbt_project.dbt_jyeo.src_utc"
03:26:26  On source.my_dbt_project.dbt_jyeo.src_utc: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "node_id": "source.my_dbt_project.dbt_jyeo.src_utc"} */
select
      max(updated_at) as max_loaded_at,
      convert_timezone('UTC', current_timestamp()) as snapshotted_at
    from development_jyeo.dbt_jyeo.src_utc
03:26:26  Opening a new connection, currently in state closed
03:26:28  SQL status: SUCCESS 1 in 2.0 seconds
03:26:28  On source.my_dbt_project.dbt_jyeo.src_utc: Close
03:26:28  3 of 3 PASS freshness of dbt_jyeo.src_utc ...................................... [PASS in 2.39s]
03:26:28  Finished running node source.my_dbt_project.dbt_jyeo.src_utc
03:26:28  Connection 'master' was properly closed.
03:26:28  Connection 'source.my_dbt_project.dbt_jyeo.src_utc' was properly closed.
03:26:28  
03:26:28  Finished running 3 sources in 0 hours 0 minutes and 7.13 seconds (7.13s).
```

As we can see from the above queries, dbt retrieve the current timestamp in UTC - this is assigned to the `snapshotted_at` variable. dbt also retrieved the `max(updated_at)` of the table of interest - this is assigned to the `max_loaded_at` variable. No matter the timestamp type or timezone, this is converted into UTC so that an accurate UTC-UTC time difference can be calculated. Next, because `max_loaded_at (in UTC) - snapshotted_at` did not exceed the 1 hour mark - dbt did not emit an error here.

Note that when dbt runs source freshness - there will be a `sources.json` artifact that's generated and there is more information in that file:

```json
    "results": [
        {
            "unique_id": "source.my_dbt_project.dbt_jyeo.src_akl",
            "max_loaded_at": "2024-07-01T03:21:22.570000+00:00",
            "snapshotted_at": "2024-07-01T03:26:23.508000+00:00",
            "max_loaded_at_time_ago_in_s": 300.938,
            "status": "pass",
            "criteria": {
                "warn_after": {
                    "count": null,
                    "period": null
                },
                "error_after": {
                    "count": 1,
                    "period": "hour"
                },
                "filter": null
            },
            ...
        },
        {
            "unique_id": "source.my_dbt_project.dbt_jyeo.src_la",
            "max_loaded_at": "2024-07-01T03:21:21.029000+00:00",
            "snapshotted_at": "2024-07-01T03:26:26.038000+00:00",
            "max_loaded_at_time_ago_in_s": 305.009,
            "status": "pass",
            "criteria": {
                "warn_after": {
                    "count": null,
                    "period": null
                },
                "error_after": {
                    "count": 1,
                    "period": "hour"
                },
                "filter": null
            },
            ...
        },
        {
            "unique_id": "source.my_dbt_project.dbt_jyeo.src_utc",
            "max_loaded_at": "2024-07-01T03:21:24.298000+00:00",
            "snapshotted_at": "2024-07-01T03:26:28.192000+00:00",
            "max_loaded_at_time_ago_in_s": 303.894,
            "status": "pass",
            "criteria": {
                "warn_after": {
                    "count": null,
                    "period": null
                },
                "error_after": {
                    "count": 1,
                    "period": "hour"
                },
                "filter": null
            },
            ...
        }
    ],
```

What do we see here? For each source table, even though the raw `updated_at` column had different timezones and timestamp types - dbt converts all of them into UTC. It then calculates the difference between that (`max_loaded_at`) and the current timestamp / `snapshotted_at` to derive `max_loaded_at_time_ago_in_s`.

And because all of the `max_loaded_at_time_ago_in_s` were roughly around `300` seconds, it is less than `3600` seconds (`1` hour) (our `error_after` property) - therefore, no errors are emitted.

Now let's change our `error_after` property so that we get an error on purpose:

```yaml
# models/sources.yml
version: 2
sources:
  - name: dbt_jyeo
    freshness:
      error_after: {count: 5, period: minute}
    loaded_at_field: update_at
    tables:
      - name: src_la
      - name: src_akl
      - name: src_utc
```

> We already know these sources were already more than 300 seconds from the previous source freshness check so by setting the `error_after` to be 5 minutes - we are expecting an error.

```sh
$ dbt --debug source freshness
03:39:26  Began running node source.my_dbt_project.dbt_jyeo.src_akl
03:39:26  1 of 3 START freshness of dbt_jyeo.src_akl ..................................... [RUN]
03:39:26  Acquiring new snowflake connection 'source.my_dbt_project.dbt_jyeo.src_akl'
03:39:26  Began compiling node source.my_dbt_project.dbt_jyeo.src_akl
03:39:26  Began executing node source.my_dbt_project.dbt_jyeo.src_akl
03:39:26  Using snowflake connection "source.my_dbt_project.dbt_jyeo.src_akl"
03:39:26  On source.my_dbt_project.dbt_jyeo.src_akl: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "node_id": "source.my_dbt_project.dbt_jyeo.src_akl"} */
select
      max(updated_at) as max_loaded_at,
      convert_timezone('UTC', current_timestamp()) as snapshotted_at
    from development_jyeo.dbt_jyeo.src_akl
03:39:26  Opening a new connection, currently in state init
03:39:29  SQL status: SUCCESS 1 in 3.0 seconds
03:39:29  On source.my_dbt_project.dbt_jyeo.src_akl: Close
03:39:29  1 of 3 ERROR STALE freshness of dbt_jyeo.src_akl ............................... [ERROR STALE in 3.37s]
03:39:29  Finished running node source.my_dbt_project.dbt_jyeo.src_akl
03:39:29  Began running node source.my_dbt_project.dbt_jyeo.src_la
03:39:29  2 of 3 START freshness of dbt_jyeo.src_la ...................................... [RUN]
03:39:29  Re-using an available connection from the pool (formerly source.my_dbt_project.dbt_jyeo.src_akl, now source.my_dbt_project.dbt_jyeo.src_la)
03:39:29  Began compiling node source.my_dbt_project.dbt_jyeo.src_la
03:39:29  Began executing node source.my_dbt_project.dbt_jyeo.src_la
03:39:29  Using snowflake connection "source.my_dbt_project.dbt_jyeo.src_la"
03:39:29  On source.my_dbt_project.dbt_jyeo.src_la: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "node_id": "source.my_dbt_project.dbt_jyeo.src_la"} */
select
      max(updated_at) as max_loaded_at,
      convert_timezone('UTC', current_timestamp()) as snapshotted_at
    from development_jyeo.dbt_jyeo.src_la
03:39:29  Opening a new connection, currently in state closed
03:39:31  SQL status: SUCCESS 1 in 2.0 seconds
03:39:31  On source.my_dbt_project.dbt_jyeo.src_la: Close
03:39:32  2 of 3 ERROR STALE freshness of dbt_jyeo.src_la ................................ [ERROR STALE in 2.45s]
03:39:32  Finished running node source.my_dbt_project.dbt_jyeo.src_la
03:39:32  Began running node source.my_dbt_project.dbt_jyeo.src_utc
03:39:32  3 of 3 START freshness of dbt_jyeo.src_utc ..................................... [RUN]
03:39:32  Re-using an available connection from the pool (formerly source.my_dbt_project.dbt_jyeo.src_la, now source.my_dbt_project.dbt_jyeo.src_utc)
03:39:32  Began compiling node source.my_dbt_project.dbt_jyeo.src_utc
03:39:32  Began executing node source.my_dbt_project.dbt_jyeo.src_utc
03:39:32  Using snowflake connection "source.my_dbt_project.dbt_jyeo.src_utc"
03:39:32  On source.my_dbt_project.dbt_jyeo.src_utc: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "node_id": "source.my_dbt_project.dbt_jyeo.src_utc"} */
select
      max(updated_at) as max_loaded_at,
      convert_timezone('UTC', current_timestamp()) as snapshotted_at
    from development_jyeo.dbt_jyeo.src_utc
03:39:32  Opening a new connection, currently in state closed
03:39:33  SQL status: SUCCESS 1 in 2.0 seconds
03:39:33  On source.my_dbt_project.dbt_jyeo.src_utc: Close
03:39:34  3 of 3 ERROR STALE freshness of dbt_jyeo.src_utc ............................... [ERROR STALE in 2.20s]
03:39:34  Finished running node source.my_dbt_project.dbt_jyeo.src_utc
03:39:34  Connection 'master' was properly closed.
03:39:34  Connection 'source.my_dbt_project.dbt_jyeo.src_utc' was properly closed.
03:39:34  
03:39:34  Finished running 3 sources in 0 hours 0 minutes and 8.04 seconds (8.04s).
```

And the `sources.json`:

```json
        {
            "unique_id": "source.my_dbt_project.dbt_jyeo.src_akl",
            "max_loaded_at": "2024-07-01T03:21:22.570000+00:00",
            "snapshotted_at": "2024-07-01T03:39:29.099000+00:00",
            "max_loaded_at_time_ago_in_s": 1086.529,
            "status": "error",
            "criteria": {
                "warn_after": {
                    "count": null,
                    "period": null
                },
                "error_after": {
                    "count": 5,
                    "period": "minute"
                },
                "filter": null
            },
            ...
        },
        {
            "unique_id": "source.my_dbt_project.dbt_jyeo.src_la",
            "max_loaded_at": "2024-07-01T03:21:21.029000+00:00",
            "snapshotted_at": "2024-07-01T03:39:31.522000+00:00",
            "max_loaded_at_time_ago_in_s": 1090.493,
            "status": "error",
            "criteria": {
                "warn_after": {
                    "count": null,
                    "period": null
                },
                "error_after": {
                    "count": 5,
                    "period": "minute"
                },
                "filter": null
            },
            ...
        },
        {
            "unique_id": "source.my_dbt_project.dbt_jyeo.src_utc",
            "max_loaded_at": "2024-07-01T03:21:24.298000+00:00",
            "snapshotted_at": "2024-07-01T03:39:33.998000+00:00",
            "max_loaded_at_time_ago_in_s": 1089.7,
            "status": "error",
            "criteria": {
                "warn_after": {
                    "count": null,
                    "period": null
                },
                "error_after": {
                    "count": 5,
                    "period": "minute"
                },
                "filter": null
            },
            ...
        }
```

The exact same thing happened here - except now, the `max_loaded_at_time_ago_in_s` all exceeded `1000` seconds which is more than the error_after property we have set `600` sceonds / `5` minutes.

These `sources.json` files can be retrieved from the "Artifacts" tab in your dbt Cloud job run or in the `target/` folder when using dbt-core CLI.
