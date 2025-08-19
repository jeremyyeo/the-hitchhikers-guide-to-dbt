---
---

# Snowflake bundle change

## 2025_03

1. The 2025_03 Bundle has been made Generally Enabled (cannot be disabled) as of August 4th 2025 - https://docs.snowflake.com/en/release-notes/bcr-bundles/2025_03_bundle
2. Along with that is a new maximum size (`134217728`) for newly created `varchar` columns - https://docs.snowflake.com/en/release-notes/bcr-bundles/2025_03/bcr-1942
3. Importantly, `varchar` columns prior to this remain it's old maximum size `16777216`.

Let's see the implications from a dbt incremental model perspective.

By default, dbt will expand an existing columns data type to accomodate a larger varchar size... let's see this in action.

First, we create an existing table:

```sql
create or replace table db.sch.foo as (
    select 1 id, 'bob'::varchar(3) as first_name
);
```

Then we build a model like so:

```sql
-- models/foo.sql
{{ config(materialized='incremental', unique_key='id') }}
select 2 id, 'alice'::varchar(5) as first_name
```

```sh
$ dbt run
00:00:43  1 of 1 START sql incremental model sch.foo ..................................... [RUN]

00:00:43  On model.analytics.foo: create or replace  temporary view db.sch.foo__dbt_tmp
  as (
select 2 id, 'alice'::varchar(5) as first_name
  )
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
00:00:44  SQL status: SUCCESS 1 in 1.668 seconds

00:00:45  On model.analytics.foo: describe table db.sch.foo__dbt_tmp
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */
00:00:45  SQL status: SUCCESS 2 in 0.739 seconds

00:00:45  On model.analytics.foo: describe table db.sch.foo
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */
00:00:46  SQL status: SUCCESS 2 in 0.941 seconds

00:00:46  Changing col type from character varying(3) to character varying(5) in table database: "db"
schema: "sch"
identifier: "foo"
00:00:46  On model.analytics.foo: alter  table db.sch.foo alter "FIRST_NAME" set data type character varying(5)
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
00:00:47  SQL status: SUCCESS 1 in 1.284 seconds

00:00:48  On model.analytics.foo: describe table "DB"."SCH"."FOO"
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */
00:00:49  SQL status: SUCCESS 2 in 1.186 seconds

00:00:49  On model.analytics.foo: -- back compat for old kwarg name
  begin
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
00:00:49  SQL status: SUCCESS 1 in 0.665 seconds
00:00:49  Using snowflake connection "model.analytics.foo"
00:00:49  On model.analytics.foo: merge into db.sch.foo as DBT_INTERNAL_DEST
        using db.sch.foo__dbt_tmp as DBT_INTERNAL_SOURCE
        on ((DBT_INTERNAL_SOURCE.id = DBT_INTERNAL_DEST.id))
    when matched then update set
        "ID" = DBT_INTERNAL_SOURCE."ID","FIRST_NAME" = DBT_INTERNAL_SOURCE."FIRST_NAME"
    when not matched then insert
        ("ID", "FIRST_NAME")
    values
        ("ID", "FIRST_NAME")
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
00:00:52  SQL status: SUCCESS 1 in 2.369 seconds

00:00:54  On model.analytics.foo: drop view if exists db.sch.foo__dbt_tmp cascade
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */
00:00:56  SQL status: SUCCESS 1 in 1.512 seconds

00:00:56  1 of 1 OK created sql incremental model sch.foo ................................ [SUCCESS 1 in 13.20s]
```

What is the sequence of operations of incremental models out of the box?

1. We create a temp table.
2. We check the data type of the columns in the temp table and the target table by running `describe table` statements.
3. We detect that (2) returns the fact that:
   1. The temp tables `first_name` column has a varchar size of (`5`).
   2. The target tables `first_name` column has a varchar size of (`3`).
4. Since `5` > `3`, we expand the existing tables varchar size `alter table db.sch.foo alter "FIRST_NAME" set data type character varying(5)`.

How would the bundle being enabled affect this behaviour?

Recall that prior to the bundle change, the max size of varchar columns was `16777216` - this means that if you had created a table with a column WITHOUT specifying the size - e.g. `select 'hello'::text as c` - then the max size of that varchar would be `16777216`. Post bundle change, the new varchar would have a size of `134217728`. Therefore without even modifying anything in your project - we would run into something like:

1. We create a temp table.
2. We check the data type of the columns in the temp table and the target table by running `describe table` statements.
3. We detect that (2) returns the fact that:
   1. The temp table has a varchar size of (`134217728`).
      - _Bundle change caused new `varchar` columns to have max size `134217728`_.
   2. The target table has a varchar size of (`16777216`).
      - _Bundle change did not retroactively go back and update old `varchar` columns to have max size `134217728`_.
4. Since `134217728` > `16777216`, we expand the existing tables varchar size `alter table db.sch.some_table alter "MY_COL" set data type character varying(134217728)`.

The implication here is that if the Snowflake role we're using does not have access to perform the `alter table ... alter col ...` operation - then your dbt job will start erroring with a Snowflake permission issue.

### Bonus implication

If you're using collation (https://docs.snowflake.com/en/sql-reference/collation) - you'll run into an additional issue:

Build the "existing table":

```sql
create or replace table db.sch.foo (
    id int,
    es varchar(6) collate 'es'
);
insert into db.sch.foo values (1, 'piñata');
```

Then run our incremental model:

```sql
-- models/foo.sql
{{ config(materialized='incremental', unique_key='id') }}
select 2 id, 'piña colada'::varchar(11) as es
```

```sh
$ dbt run

01:03:56  1 of 1 START sql incremental model sch.foo ..................................... [RUN]
01:03:56  Re-using an available connection from the pool (formerly list_db_sch, now model.analytics.foo)
01:03:56  Began compiling node model.analytics.foo
01:03:56  Writing injected SQL for node "model.analytics.foo"
01:03:56  Began executing node model.analytics.foo
01:03:56  Using snowflake connection "model.analytics.foo"
01:03:56  On model.analytics.foo: create or replace  temporary view db.sch.foo__dbt_tmp

  as (
    -- models/foo.sql

select 2 id, 'piña colada'::varchar(11) as es
  )
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
01:03:56  SQL status: SUCCESS 1 in 0.360 seconds
01:03:56  Using snowflake connection "model.analytics.foo"
01:03:56  On model.analytics.foo: describe table db.sch.foo__dbt_tmp
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */
01:03:57  SQL status: SUCCESS 2 in 0.342 seconds
01:03:57  Using snowflake connection "model.analytics.foo"
01:03:57  On model.analytics.foo: describe table db.sch.foo
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */
01:03:57  SQL status: SUCCESS 2 in 0.299 seconds
01:03:57  Changing col type from character varying(6) to character varying(11) in table database: "db"
schema: "sch"
identifier: "foo"

01:03:57  Using snowflake connection "model.analytics.foo"
01:03:57  On model.analytics.foo: alter  table db.sch.foo alter "ES" set data type character varying(11)
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
01:03:57  Snowflake adapter: Snowflake query id: 01be769f-0709-3903-000d-378351d13c9e
01:03:57  Snowflake adapter: Snowflake error: 040053 (22000): SQL compilation error: cannot change column ES from type "VARCHAR(6) COLLATE 'es'" to "VARCHAR(11)" because they have incompatible collations.
01:03:57  Snowflake adapter: Error running SQL: macro alter_column_type
01:03:57  Snowflake adapter: Rolling back transaction.
01:03:57  Database Error in model foo (models/foo.sql)
  040053 (22000): SQL compilation error: cannot change column ES from type "VARCHAR(6) COLLATE 'es'" to "VARCHAR(11)" because they have incompatible collations.
01:03:57  1 of 1 ERROR creating sql incremental model sch.foo ............................ [ERROR in 1.37s]
```

We run through the same sequence of operations as before... dbt emits `alter  table db.sch.foo alter "ES" set data type character varying(11)` but it errors because it needs extra collate keywords:

```sql
alter  table db.sch.foo alter "ES" set data type character varying(11) collate 'es';
```

dbt currently does not go beyond the data type (i.e. `varchar`) and it's size (i.e. `11`). The `collate 'es'` option on is transparent to dbt. We can demonstrate this very quickly:

```sql
-- macros/check.sql
{% macro check() %}
    {% set columns = adapter.get_columns_in_relation(ref('foo')) %}
    {% do print(columns) %}
{% endmacro %}
```

```sh
$ dbt --debug run-operation check

01:10:52  On macro_check: describe table db.sch.foo
/* {"app": "dbt", "dbt_version": "1.10.9", "profile_name": "sf", "target_name": "ci", "connection_name": "macro_check"} */
01:10:53  SQL status: SUCCESS 2 in 1.593 seconds

[SnowflakeColumn(column='ID', dtype='NUMBER', char_size=None, numeric_precision=38, numeric_scale=0), SnowflakeColumn(column='ES', dtype='VARCHAR', char_size=11, numeric_precision=None, numeric_scale=None)]
```

^ As we can see, we didn't retrieve the `COLLATE 'es'` option on the `es` column.
