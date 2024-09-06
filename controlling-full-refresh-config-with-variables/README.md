---
---

## Controlling full refresh config with variables

Some folks want to set the `full_refresh` config dynamically - i.e.

```sql
-- models/foo.sql
{{ 
    config(
        materialized='incremental',
        full_refresh='{{ macro_that_runs_some_sql_query_first() }}'
    ) 
}}
...
```

It is however not possible to set configs based on the outcome of a SQL query: https://github.com/dbt-labs/docs.getdbt.com/discussions/1310.

We can use vars to control whether a model should have it's full_refresh config be set to `true` or `false` like so:

```sql
-- models/foo.sql
{{
    config(
        materialized="incremental",
        full_refresh=(this.name in var('allow_full_refresh', [])) 
    )
}}
select 1 id

-- models/bar.sql
{{
    config(
        materialized="incremental",
        full_refresh=(this.name in var('allow_full_refresh', [])) 
    )
}}
select 1 id
```

Let's see this in action... first we delete all the existing tables.

```sql
drop table if exists db.analytics.foo;
drop table if exists db.analytics.bar;
```

> Logs below are truncated just to show what is necessary.

Then do our first run:

```sh
$ dbt run

21:44:20  On model.my_dbt_project.bar: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.bar"} */
create or replace transient table db.analytics.bar
         as
        (

select 1 id
        );

21:44:23  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table db.analytics.foo
         as
        (

select 1 id
        );
```

Table is created from scratch as expected. Now we do a subsequent run:

```sh
$ dbt run

21:45:32  On model.my_dbt_project.bar: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.bar"} */
insert into db.analytics.bar ("ID")
        (
            select "ID"
            from db.analytics.bar__dbt_tmp
        );

21:45:36  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
insert into db.analytics.foo ("ID")
        (
            select "ID"
            from db.analytics.foo__dbt_tmp
        );
```

`foo` and `bar` were inserted into as expected of an incremental subsequent run. Let's try to add our `--full-refresh` flag and see:

```sh
$ dbt run --full-refresh

21:46:56  On model.my_dbt_project.bar: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.bar"} */
insert into db.analytics.bar ("ID")
        (
            select "ID"
            from db.analytics.bar__dbt_tmp
        );

21:47:00  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
insert into db.analytics.foo ("ID")
        (
            select "ID"
            from db.analytics.foo__dbt_tmp
        );
```

Even with the `--full-refresh` flag, those models were NOT fully refreshed cause their `full_refresh` configs resolved to `false`. Let's set the var so that model `bar` will full refresh but not `foo`:

```sh
$ dbt run --full-refresh --vars 'allow_full_refresh: ["bar"]'

21:49:18  On model.my_dbt_project.bar: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.bar"} */
create or replace transient table db.analytics.bar
         as
        (

select 1 id
        );

21:49:23  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
insert into db.analytics.foo ("ID")
        (
            select "ID"
            from db.analytics.foo__dbt_tmp
        );
```

By setting the vars accordingly we can prevent or allow models from fully refreshing whenever we run with the `--full-refresh` flag.
