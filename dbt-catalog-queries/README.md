---
---

# dbt catalog queries

## Debugging the catalog queries

When `dbt docs generate` runs, dbt will retrieve metadata about all the tables in all the relevant databases and schemas that are in your project. If we find any issues with the generated catalog files, we can try and debug this by finding out what those catalog queries actually return when they run. By default, dbt does not log out the results of running the catalog queries so will have to make dbt do exactly that.

First, we should try and have a look at all the catalog queries that run in the debug logs - here's an example from a recent run:

```sh
$ dbt docs generate
...
02:54:16  Building catalog
02:54:16  Acquiring new snowflake connection 'db.information_schema'
02:54:16  Using snowflake connection "db.information_schema"
02:54:16  On db.information_schema: /* {"app": "dbt", "dbt_version": "1.9.2", "profile_name": "sf", "target_name": "ci", "connection_name": "db.information_schema"} */
with tables as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        case
            when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
            else table_type
        end as "table_type",
        comment as "table_comment",

        -- note: this is the _role_ that owns the table
        table_owner as "table_owner",

        'Clustering Key' as "stats:clustering_key:label",
        clustering_key as "stats:clustering_key:value",
        'The key used to cluster this table' as "stats:clustering_key:description",
        (clustering_key is not null) as "stats:clustering_key:include",

        'Row Count' as "stats:row_count:label",
        row_count as "stats:row_count:value",
        'An approximate count of rows in this table' as "stats:row_count:description",
        (row_count is not null) as "stats:row_count:include",

        'Approximate Size' as "stats:bytes:label",
        bytes as "stats:bytes:value",
        'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
        (bytes is not null) as "stats:bytes:include",

        'Last Modified' as "stats:last_modified:label",
        to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
        'The timestamp for last update/change' as "stats:last_modified:description",
        (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
    from db.INFORMATION_SCHEMA.tables
            where ((
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        ),
        columns as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",

        column_name as "column_name",
        ordinal_position as "column_index",
        data_type as "column_type",
        comment as "column_comment"
    from db.INFORMATION_SCHEMA.columns
            where ((
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        )
        select *
    from tables
    join columns using ("table_database", "table_schema", "table_name")
    order by "column_index"
02:54:16  Opening a new connection, currently in state init
02:54:19  SQL status: SUCCESS 110 in 3.462 seconds
02:54:19  Wrote artifact CatalogArtifact to /Users/jeremy/git/dbt-basic/target/catalog.json
02:54:20  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
02:54:20  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
02:54:20  Catalog written to /Users/jeremy/git/dbt-basic/target/catalog.json
02:54:20  Resource report: {"command_name": "generate", "command_success": true, "command_wall_clock_time": 6.9147506, "process_in_blocks": "0", "process_kernel_time": 0.22012, "process_mem_max_rss": "204013568", "process_out_blocks": "0", "process_user_time": 1.446297}
02:54:20  Command `dbt docs generate` succeeded at 15:54:20.014979 after 6.92 seconds
...
```

What we can do is add that query into a macro that we can call in an on-run-x hook or even a separate run-operation:

```sql
-- macros/check_catalog.sql
{% macro check_catalog() %}
   {% set query %}
        with tables as (
                    select
                table_catalog as "table_database",
                table_schema as "table_schema",
                table_name as "table_name",
                case
                    when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
                    else table_type
                end as "table_type",
                comment as "table_comment",

                -- note: this is the _role_ that owns the table
                table_owner as "table_owner",

                'Clustering Key' as "stats:clustering_key:label",
                clustering_key as "stats:clustering_key:value",
                'The key used to cluster this table' as "stats:clustering_key:description",
                (clustering_key is not null) as "stats:clustering_key:include",

                'Row Count' as "stats:row_count:label",
                row_count as "stats:row_count:value",
                'An approximate count of rows in this table' as "stats:row_count:description",
                (row_count is not null) as "stats:row_count:include",

                'Approximate Size' as "stats:bytes:label",
                bytes as "stats:bytes:value",
                'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
                (bytes is not null) as "stats:bytes:include",

                'Last Modified' as "stats:last_modified:label",
                to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
                'The timestamp for last update/change' as "stats:last_modified:description",
                (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
            from db.INFORMATION_SCHEMA.tables
                    where ((
            "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
        ))
                ),
                columns as (
                    select
                table_catalog as "table_database",
                table_schema as "table_schema",
                table_name as "table_name",

                column_name as "column_name",
                ordinal_position as "column_index",
                data_type as "column_type",
                comment as "column_comment"
            from db.INFORMATION_SCHEMA.columns
                    where ((
            "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
        ))
                )
                select *
            from tables
            join columns using ("table_database", "table_schema", "table_name")
            order by "column_index"
   {% endset %}

   {% if execute %}
    {% set res = run_query(query) %}
    {% for row in res %}
        {% do print(dict(row)) %}
    {% endfor %}
   {% endif %}
{% endmacro %}
```

```yaml
# dbt_project.yml
---
on-run-end: "{% if flags.WHICH == 'generate' %}{{ check_catalog() }}{% endif %}"
```

^ In my example here, I'm going to run that macro in an on-run-end hook and only if the command was `dbt docs generate`.

```sh
$ dbt docs generate
...
03:02:04  Began compiling node operation.my_dbt_project.my_dbt_project-on-run-end-0
03:02:04  Using snowflake connection "operation.my_dbt_project.my_dbt_project-on-run-end-0"
03:02:04  On operation.my_dbt_project.my_dbt_project-on-run-end-0: /* {"app": "dbt", "dbt_version": "1.9.2", "profile_name": "sf", "target_name": "ci", "node_id": "operation.my_dbt_project.my_dbt_project-on-run-end-0"} */
with tables as (
                    select
                table_catalog as "table_database",
                table_schema as "table_schema",
                table_name as "table_name",
                case
                    when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
                    else table_type
                end as "table_type",
                comment as "table_comment",

                -- note: this is the _role_ that owns the table
                table_owner as "table_owner",

                'Clustering Key' as "stats:clustering_key:label",
                clustering_key as "stats:clustering_key:value",
                'The key used to cluster this table' as "stats:clustering_key:description",
                (clustering_key is not null) as "stats:clustering_key:include",

                'Row Count' as "stats:row_count:label",
                row_count as "stats:row_count:value",
                'An approximate count of rows in this table' as "stats:row_count:description",
                (row_count is not null) as "stats:row_count:include",

                'Approximate Size' as "stats:bytes:label",
                bytes as "stats:bytes:value",
                'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
                (bytes is not null) as "stats:bytes:include",

                'Last Modified' as "stats:last_modified:label",
                to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
                'The timestamp for last update/change' as "stats:last_modified:description",
                (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
            from db.INFORMATION_SCHEMA.tables
                    where ((
            "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
        ))
                ),
                columns as (
                    select
                table_catalog as "table_database",
                table_schema as "table_schema",
                table_name as "table_name",

                column_name as "column_name",
                ordinal_position as "column_index",
                data_type as "column_type",
                comment as "column_comment"
            from db.INFORMATION_SCHEMA.columns
                    where ((
            "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
        ))
                )
                select *
            from tables
            join columns using ("table_database", "table_schema", "table_name")
            order by "column_index"
03:02:07  SQL status: SUCCESS 110 in 2.764 seconds
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00047', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00018', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00085', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00037', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00053', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00031', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00105', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00032', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00046', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00103', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00076', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00041', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00022', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00010', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00040', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00065', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00082', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00025', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00107', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00106', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00097', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00108', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00048', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00058', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00029', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00001', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00100', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00027', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00014', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00039', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00102', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00093', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00099', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00096', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00098', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00011', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00013', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00109', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00079', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00071', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00070', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00068', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00089', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00016', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00072', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00057', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00021', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00054', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00008', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00004', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00059', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00007', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00051', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00078', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00063', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00049', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00075', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00086', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00104', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00087', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00005', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00062', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00002', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00000', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00026', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00045', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00036', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00090', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00067', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00033', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00083', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00101', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00091', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00020', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00034', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00061', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00024', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00081', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00080', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00064', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00015', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00009', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00030', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00028', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00019', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00077', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00012', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00095', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00017', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00084', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00003', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00073', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00043', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00056', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00035', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00050', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00060', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00066', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00006', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00055', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00088', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00038', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00044', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00092', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00052', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00094', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00042', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00074', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00069', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
{'table_database': 'DB', 'table_schema': 'SCH', 'table_name': 'FOO_00023', 'table_type': 'BASE TABLE', 'table_comment': None, 'table_owner': 'TRANSFORMER', 'stats:clustering_key:label': 'Clustering Key', 'stats:clustering_key:value': None, 'stats:clustering_key:description': 'The key used to cluster this table', 'stats:clustering_key:include': False, 'stats:row_count:label': 'Row Count', 'stats:row_count:value': 1, 'stats:row_count:description': 'An approximate count of rows in this table', 'stats:row_count:include': True, 'stats:bytes:label': 'Approximate Size', 'stats:bytes:value': 1024, 'stats:bytes:description': 'Approximate size of the table as reported by Snowflake', 'stats:bytes:include': True, 'stats:last_modified:label': 'Last Modified', 'stats:last_modified:value': '2025-02-14 02:59UTC', 'stats:last_modified:description': 'The timestamp for last update/change', 'stats:last_modified:include': True, 'column_name': 'ID', 'column_index': 1, 'column_type': 'NUMBER', 'column_comment': None}
03:02:07  Writing injected SQL for node "operation.my_dbt_project.my_dbt_project-on-run-end-0"
03:02:07  Began executing node operation.my_dbt_project.my_dbt_project-on-run-end-0
03:02:07  Finished running node operation.my_dbt_project.my_dbt_project-on-run-end-0
03:02:07  Connection 'master' was properly closed.
03:02:07  Connection 'operation.my_dbt_project.my_dbt_project-on-run-end-0' was left open.
03:02:07  On operation.my_dbt_project.my_dbt_project-on-run-end-0: Close
03:02:08  Command end result
03:02:08  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
03:02:08  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
03:02:08  Wrote artifact RunExecutionResult to /Users/jeremy/git/dbt-basic/target/run_results.json
03:02:08  Acquiring new snowflake connection 'generate_catalog'
03:02:08  Building catalog
03:02:08  Acquiring new snowflake connection 'db.information_schema'
03:02:08  Using snowflake connection "db.information_schema"
03:02:08  On db.information_schema: /* {"app": "dbt", "dbt_version": "1.9.2", "profile_name": "sf", "target_name": "ci", "connection_name": "db.information_schema"} */
with tables as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        case
            when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
            else table_type
        end as "table_type",
        comment as "table_comment",

        -- note: this is the _role_ that owns the table
        table_owner as "table_owner",

        'Clustering Key' as "stats:clustering_key:label",
        clustering_key as "stats:clustering_key:value",
        'The key used to cluster this table' as "stats:clustering_key:description",
        (clustering_key is not null) as "stats:clustering_key:include",

        'Row Count' as "stats:row_count:label",
        row_count as "stats:row_count:value",
        'An approximate count of rows in this table' as "stats:row_count:description",
        (row_count is not null) as "stats:row_count:include",

        'Approximate Size' as "stats:bytes:label",
        bytes as "stats:bytes:value",
        'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
        (bytes is not null) as "stats:bytes:include",

        'Last Modified' as "stats:last_modified:label",
        to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
        'The timestamp for last update/change' as "stats:last_modified:description",
        (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
    from db.INFORMATION_SCHEMA.tables
            where ((
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        ),
        columns as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",

        column_name as "column_name",
        ordinal_position as "column_index",
        data_type as "column_type",
        comment as "column_comment"
    from db.INFORMATION_SCHEMA.columns
            where ((
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        )
        select *
    from tables
    join columns using ("table_database", "table_schema", "table_name")
    order by "column_index"
03:02:08  Opening a new connection, currently in state init
03:02:11  SQL status: SUCCESS 110 in 3.759 seconds
03:02:11  Wrote artifact CatalogArtifact to /Users/jeremy/git/dbt-basic/target/catalog.json
03:02:11  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
03:02:11  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
03:02:11  Catalog written to /Users/jeremy/git/dbt-basic/target/catalog.json
03:02:11  Resource report: {"command_name": "generate", "command_success": true, "command_wall_clock_time": 10.442908, "process_in_blocks": "0", "process_kernel_time": 0.28265, "process_mem_max_rss": "205701120", "process_out_blocks": "0", "process_user_time": 1.53923}
03:02:11  Command `dbt docs generate` succeeded at 16:02:11.914756 after 10.44 seconds
```

With that in place, we would log the results of the catalog queries so we can debug appropriately. Note that this is just a simple example with Snowflake, you will need to adapt the code to your own dwh and depending on the number of schemas and databases that exist in your project, you may have very many catalog queries to execute as opposed to just the single one shown above. The more advanced amongst you would be able to override the built-in macros to do the same thing in a more dynamic manner https://github.com/dbt-labs/dbt-adapters/blob/main/dbt-snowflake/src/dbt/include/snowflake/macros/catalog.sql

## Limiting the catalog queries

> Snowflake example follows but the principle can be applied to any adapter.

When we invoke `dbt docs generate`, dbt will run introspective queries that reads table/column metadata from the databases information schema meta tables / views. Let's see this in action:

> The default schema on the connection I'm using with my project below is `sch`.

```yaml
# dbt_project.yml
name: analytics
profile: all
version: "1.0.0"

models:
  analytics:
    +materialized: table
    marts:
      +database: db
    staging:
      +database: database_no_access

sources:
  - name: raw
    database: database_no_access
    schema: schema_no_access
    tables:
      - name: customers
```

> The database configs of `database_no_access` is chosen to demonstrate that the role we're using has NO permissions to this database.

```sql
-- models/marts/foo.sql
select 1 id

-- models/staging/bar.sql
select * from {{ source('raw', 'customers') }}
```

```sh
$ dbt --debug docs generate
...
01:52:23  Acquiring new snowflake connection 'generate_catalog'
01:52:23  Building catalog
01:52:23  Acquiring new snowflake connection 'db.information_schema'
01:52:23  Using snowflake connection "db.information_schema"
01:52:23  On db.information_schema: with tables as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        case
            when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
            else table_type
        end as "table_type",
        comment as "table_comment",
        -- note: this is the _role_ that owns the table
        table_owner as "table_owner",
        'Clustering Key' as "stats:clustering_key:label",
        clustering_key as "stats:clustering_key:value",
        'The key used to cluster this table' as "stats:clustering_key:description",
        (clustering_key is not null) as "stats:clustering_key:include",
        'Row Count' as "stats:row_count:label",
        row_count as "stats:row_count:value",
        'An approximate count of rows in this table' as "stats:row_count:description",
        (row_count is not null) as "stats:row_count:include",
        'Approximate Size' as "stats:bytes:label",
        bytes as "stats:bytes:value",
        'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
        (bytes is not null) as "stats:bytes:include",
        'Last Modified' as "stats:last_modified:label",
        to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
        'The timestamp for last update/change' as "stats:last_modified:description",
        (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
    from db.INFORMATION_SCHEMA.tables
            where (
                (
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
                    and
    "table_name" ilike 'foo' and upper("table_name") = upper('foo')
                )
            )
        ),
        columns as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",

        column_name as "column_name",
        ordinal_position as "column_index",
        data_type as "column_type",
        comment as "column_comment"
    from db.INFORMATION_SCHEMA.columns
            where (
                (
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
                    and
    "table_name" ilike 'foo' and upper("table_name") = upper('foo')
                )
            )
        )
        select *
    from tables
    join columns using ("table_database", "table_schema", "table_name")
    order by "column_index"
/* {"app": "dbt", "dbt_version": "1.11.0b3", "profile_name": "all", "target_name": "sf", "connection_name": "db.information_schema"} */
01:52:23  Opening a new connection, currently in state init
01:52:27  SQL status: SUCCESS 1 in 3.682 seconds
01:52:27  Re-using an available connection from the pool (formerly db.information_schema, now database_no_access.information_schema)
01:52:27  Using snowflake connection "database_no_access.information_schema"
01:52:27  On database_no_access.information_schema: with tables as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        case
            when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
            else table_type
        end as "table_type",
        comment as "table_comment",
        -- note: this is the _role_ that owns the table
        table_owner as "table_owner",
        'Clustering Key' as "stats:clustering_key:label",
        clustering_key as "stats:clustering_key:value",
        'The key used to cluster this table' as "stats:clustering_key:description",
        (clustering_key is not null) as "stats:clustering_key:include",
        'Row Count' as "stats:row_count:label",
        row_count as "stats:row_count:value",
        'An approximate count of rows in this table' as "stats:row_count:description",
        (row_count is not null) as "stats:row_count:include",
        'Approximate Size' as "stats:bytes:label",
        bytes as "stats:bytes:value",
        'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
        (bytes is not null) as "stats:bytes:include",
        'Last Modified' as "stats:last_modified:label",
        to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
        'The timestamp for last update/change' as "stats:last_modified:description",
        (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
    from database_no_access.INFORMATION_SCHEMA.tables
            where (
                (
    "table_schema" ilike 'schema_no_access' and upper("table_schema") = upper('schema_no_access')
                    and
    "table_name" ilike 'customers' and upper("table_name") = upper('customers')
                )
             or
                (
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
                    and
    "table_name" ilike 'bar' and upper("table_name") = upper('bar')
                )
            )
        ),
        columns as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        column_name as "column_name",
        ordinal_position as "column_index",
        data_type as "column_type",
        comment as "column_comment"
    from database_no_access.INFORMATION_SCHEMA.columns
            where (
                (
    "table_schema" ilike 'schema_no_access' and upper("table_schema") = upper('schema_no_access')
                    and
    "table_name" ilike 'customers' and upper("table_name") = upper('customers')
                )
             or
                (
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
                    and
    "table_name" ilike 'bar' and upper("table_name") = upper('bar')
                )
            )
        )
        select *
    from tables
    join columns using ("table_database", "table_schema", "table_name")
    order by "column_index"
/* {"app": "dbt", "dbt_version": "1.11.0b3", "profile_name": "all", "target_name": "sf", "connection_name": "database_no_access.information_schema"} */
01:52:27  Snowflake adapter: Snowflake query id: 01bfded0-0609-c8ba-000d-3783565defba
01:52:27  Snowflake adapter: Snowflake error: 002003 (02000): SQL compilation error:
Database 'DATABASE_NO_ACCESS' does not exist or not authorized.
01:52:27  Snowflake adapter: Error running SQL: macro get_catalog_relations
01:52:27  Snowflake adapter: Rolling back transaction.
01:52:27  Encountered an error while generating catalog: Database Error
  002003 (02000): SQL compilation error:
  Database 'DATABASE_NO_ACCESS' does not exist or not authorized.
01:52:27  Wrote artifact CatalogArtifact to /Users/jeremy/git/dbt-basic/target/catalog.json
01:52:27  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
01:52:27  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
01:52:27  dbt encountered 1 failure while writing the catalog
01:52:27  Catalog written to /Users/jeremy/git/dbt-basic/target/catalog.json
01:52:27  Resource report: {"command_name": "generate", "command_success": false, "command_wall_clock_time": 10.486836, "process_in_blocks": "0", "process_kernel_time": 0.285751, "process_mem_max_rss": "232996864", "process_out_blocks": "0", "process_user_time": 1.593965}
01:52:27  Command `dbt docs generate` failed at 14:52:27.471170 after 10.49 seconds
01:52:27  Connection 'generate_catalog' was properly closed.
01:52:27  Connection 'database_no_access.information_schema' was left open.
01:52:27  On database_no_access.information_schema: Close
```

What we see here is that for a blanket `dbt docs generate` command - dbt tried to introspect all tables and schema that are part of the project - and that includes peering into the database `database_no_access`'s information schema. This then results in the dbt job erroring. Some users next learn that the docs generate command can make use of `--select`. So let's say we want to only limit it to models in the `marts/` folder - as the database `db` is the only database that our role has access to.

```sh
$ dbt --debug docs generate -s 'path:models/marts'
...
02:00:26  Acquiring new snowflake connection 'generate_catalog'
02:00:26  Building catalog
02:00:26  Acquiring new snowflake connection 'db.information_schema'
02:00:26  Using snowflake connection "db.information_schema"
02:00:26  On db.information_schema: with tables as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        case
            when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
            else table_type
        end as "table_type",
        comment as "table_comment",
        -- note: this is the _role_ that owns the table
        table_owner as "table_owner",
        'Clustering Key' as "stats:clustering_key:label",
        clustering_key as "stats:clustering_key:value",
        'The key used to cluster this table' as "stats:clustering_key:description",
        (clustering_key is not null) as "stats:clustering_key:include",
        'Row Count' as "stats:row_count:label",
        row_count as "stats:row_count:value",
        'An approximate count of rows in this table' as "stats:row_count:description",
        (row_count is not null) as "stats:row_count:include",
        'Approximate Size' as "stats:bytes:label",
        bytes as "stats:bytes:value",
        'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
        (bytes is not null) as "stats:bytes:include",

        'Last Modified' as "stats:last_modified:label",
        to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
        'The timestamp for last update/change' as "stats:last_modified:description",
        (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
    from db.INFORMATION_SCHEMA.tables
            where (
                (
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
                    and
    "table_name" ilike 'foo' and upper("table_name") = upper('foo')

                )
            )
        ),
        columns as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",

        column_name as "column_name",
        ordinal_position as "column_index",
        data_type as "column_type",
        comment as "column_comment"
    from db.INFORMATION_SCHEMA.columns
            where (
                (
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
                    and
    "table_name" ilike 'foo' and upper("table_name") = upper('foo')
                )
            )
        )
        select *
    from tables
    join columns using ("table_database", "table_schema", "table_name")
    order by "column_index"
/* {"app": "dbt", "dbt_version": "1.11.0b3", "profile_name": "all", "target_name": "sf", "connection_name": "db.information_schema"} */
02:00:26  Opening a new connection, currently in state init
02:00:31  SQL status: SUCCESS 1 in 5.427 seconds
02:00:31  Wrote artifact CatalogArtifact to /Users/jeremy/git/dbt-basic/target/catalog.json
02:00:31  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
02:00:31  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
02:00:31  Catalog written to /Users/jeremy/git/dbt-basic/target/catalog.json
02:00:31  Resource report: {"command_name": "generate", "command_success": true, "command_wall_clock_time": 11.988735, "process_in_blocks": "0", "process_kernel_time": 0.339353, "process_mem_max_rss": "228229120", "process_out_blocks": "0", "process_user_time": 1.429887}
02:00:31  Command `dbt docs generate` succeeded at 15:00:31.789200 after 11.99 seconds
02:00:31  Connection 'generate_catalog' was properly closed.
02:00:31  Connection 'db.information_schema' was left open.
02:00:31  On db.information_schema: Close
```

^ Here we see that dbt rightly only introspects the database `db` and does not attempt to introspect the database `db_no_access` (since no nodes are part of the selection `-s 'path:models/marts'`). Great.

Now, let's add 100 additional models into the `marts` folder...

```sql
-- models/marts/hundred_models/foo_000.sql
select 1 id
-- models/marts/hundred_models/foo_001.sql
select 1 id
-- models/marts/hundred_models/foo_<002...098>.sql
select 1 id
-- models/marts/hundred_models/foo_099.sql
select 1 id
```

And then lets rerun our docs generate with selection:

```sh
$ dbt --debug docs generate -s 'path:models/marts'
...
02:08:00  Building catalog
02:08:00  Acquiring new snowflake connection 'db.information_schema'
02:08:00  Using snowflake connection "db.information_schema"
02:08:00  On db.information_schema: with tables as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        case
            when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
            else table_type
        end as "table_type",
        comment as "table_comment",
        -- note: this is the _role_ that owns the table
        table_owner as "table_owner",
        'Clustering Key' as "stats:clustering_key:label",
        clustering_key as "stats:clustering_key:value",
        'The key used to cluster this table' as "stats:clustering_key:description",
        (clustering_key is not null) as "stats:clustering_key:include",
        'Row Count' as "stats:row_count:label",
        row_count as "stats:row_count:value",
        'An approximate count of rows in this table' as "stats:row_count:description",
        (row_count is not null) as "stats:row_count:include",
        'Approximate Size' as "stats:bytes:label",
        bytes as "stats:bytes:value",
        'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
        (bytes is not null) as "stats:bytes:include",

        'Last Modified' as "stats:last_modified:label",
        to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
        'The timestamp for last update/change' as "stats:last_modified:description",
        (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
    from db.INFORMATION_SCHEMA.tables
            where ((
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        ),
        columns as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        column_name as "column_name",
        ordinal_position as "column_index",
        data_type as "column_type",
        comment as "column_comment"
    from db.INFORMATION_SCHEMA.columns
            where ((
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        )
        select *
    from tables
    join columns using ("table_database", "table_schema", "table_name")
    order by "column_index"
/* {"app": "dbt", "dbt_version": "1.11.0b3", "profile_name": "all", "target_name": "sf", "connection_name": "db.information_schema"} */
02:08:00  Opening a new connection, currently in state init
02:08:06  SQL status: SUCCESS 530 in 6.461 seconds
02:08:08  Re-using an available connection from the pool (formerly db.information_schema, now database_no_access.information_schema)
02:08:08  Using snowflake connection "database_no_access.information_schema"
02:08:08  On database_no_access.information_schema: with tables as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        case
            when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
            else table_type
        end as "table_type",
        comment as "table_comment",
        -- note: this is the _role_ that owns the table
        table_owner as "table_owner",
        'Clustering Key' as "stats:clustering_key:label",
        clustering_key as "stats:clustering_key:value",
        'The key used to cluster this table' as "stats:clustering_key:description",
        (clustering_key is not null) as "stats:clustering_key:include",
        'Row Count' as "stats:row_count:label",
        row_count as "stats:row_count:value",
        'An approximate count of rows in this table' as "stats:row_count:description",
        (row_count is not null) as "stats:row_count:include",
        'Approximate Size' as "stats:bytes:label",
        bytes as "stats:bytes:value",
        'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
        (bytes is not null) as "stats:bytes:include",
        'Last Modified' as "stats:last_modified:label",
        to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
        'The timestamp for last update/change' as "stats:last_modified:description",
        (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
    from database_no_access.INFORMATION_SCHEMA.tables
            where ((
    "table_schema" ilike 'schema_no_access' and upper("table_schema") = upper('schema_no_access')
) or (
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        ),
        columns as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        column_name as "column_name",
        ordinal_position as "column_index",
        data_type as "column_type",
        comment as "column_comment"
    from database_no_access.INFORMATION_SCHEMA.columns
            where ((
    "table_schema" ilike 'schema_no_access' and upper("table_schema") = upper('schema_no_access')
) or (
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        )
        select *
    from tables
    join columns using ("table_database", "table_schema", "table_name")
    order by "column_index"
/* {"app": "dbt", "dbt_version": "1.11.0b3", "profile_name": "all", "target_name": "sf", "connection_name": "database_no_access.information_schema"} */
02:08:08  Snowflake adapter: Snowflake query id: 01bfdee0-0609-c820-000d-3783565e79e2
02:08:08  Snowflake adapter: Snowflake error: 002003 (02000): SQL compilation error:
Database 'DATABASE_NO_ACCESS' does not exist or not authorized.
02:08:08  Snowflake adapter: Error running SQL: macro get_catalog
02:08:08  Snowflake adapter: Rolling back transaction.
02:08:08  Encountered an error while generating catalog: Database Error
  002003 (02000): SQL compilation error:
  Database 'DATABASE_NO_ACCESS' does not exist or not authorized.
02:08:08  Wrote artifact CatalogArtifact to /Users/jeremy/git/dbt-basic/target/catalog.json
02:08:08  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
02:08:08  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
02:08:08  dbt encountered 1 failure while writing the catalog
02:08:08  Catalog written to /Users/jeremy/git/dbt-basic/target/catalog.json
02:08:08  Resource report: {"command_name": "generate", "command_success": false, "command_wall_clock_time": 15.441701, "process_in_blocks": "0", "process_kernel_time": 0.351577, "process_mem_max_rss": "237387776", "process_out_blocks": "0", "process_user_time": 2.073535}
02:08:08  Command `dbt docs generate` failed at 15:08:08.678349 after 15.44 seconds
02:08:08  Connection 'generate_catalog' was properly closed.
02:08:08  Connection 'database_no_access.information_schema' was left open.
02:08:08  On database_no_access.information_schema: Close
```

What we notice here is that dbt has gone back to it's old ways of attempting to introspect ALL THE THINGS - which includes also attempting to query `db_no_access`'s information schema. What happens here is that when the node count exceeds 100, db will query the information schema of ALL databases that exist in the dbt project - and then only filter the results to the selected nodes AFTER THE FACT. You can read more about this (https://github.com/dbt-labs/dbt-core/issues/9506, https://github.com/dbt-labs/dbt-core/issues/9394).

What can we do about this? We can override the out of the box `get_catalog` macro and add more logic to it. First lets add the following macro to the project:

```sql
-- macros/get_catalog.sql
{% macro get_catalog(information_schema, schemas) -%}

    {% set placeholder_query -%}
        with tables as (
          select
            'PLACEHOLDER' as "table_database",
            'PLACEHOLDER' as "table_schema",
            'PLACEHOLDER' as "table_name",
            'PLACEHOLDER' as "table_type",
            null as "table_comment",
            'PLACEHOLDER' as "table_owner",
            'PLACEHOLDER' as "stats:clustering_key:label",
            null as "stats:clustering_key:value",
            'The key used to cluster this table' as "stats:clustering_key:description",
            false as "stats:clustering_key:include",
            'Row Count' as "stats:row_count:label",
            null as "stats:row_count:value",
            'An approximate count of rows in this table' as "stats:row_count:description",
            null as "stats:row_count:include",
            'Approximate Size' as "stats:bytes:label",
            null as "stats:bytes:value",
            'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
            false as "stats:bytes:include",
            'Last Modified' as "stats:last_modified:label",
            to_varchar(current_timestamp, 'yyyy-mm-dd HH24:MI' || 'UTC') as "stats:last_modified:value",
            'The timestamp for last update/change' as "stats:last_modified:description",
            false as "stats:last_modified:include"
        ),
        columns as (
            select
              'PLACEHOLDER' as "table_database",
              'PLACEHOLDER' as "table_schema",
              'PLACEHOLDER' as "table_name",
              'PLACEHOLDER' as "column_name",
              1 as "column_index",
              'TEXT' as "column_type",
              null as "column_comment"
        )
        select *
          from tables
          join columns using ("table_database", "table_schema", "table_name")
        /* Placeholder query replacing the cataloging of database '{{ information_schema.database }}' to prevent errors during docs generate with selection. */
        /* Adapted from https://github.com/dbt-labs/dbt-adapters/blob/17c948802a5b4c194354225f76977d78d0fb75c6/dbt-snowflake/src/dbt/include/snowflake/macros/catalog.sql */
    {%- endset %}

    {% if var('catalog_db_include', false) %}
        {% if information_schema.database in var('catalog_db_include') %}
            {{ return(adapter.dispatch('get_catalog', 'dbt')(information_schema, schemas)) }}
        {% else %}
            {{ return(run_query(placeholder_query)) }}
        {% endif %}
    {% else %}
        {{ return(adapter.dispatch('get_catalog', 'dbt')(information_schema, schemas)) }}
    {% endif %}

{%- endmacro %}
```

We can see the logic here is to run some sort of placeholder query that will simply run a select statement with dummy results if a certain var is included.

```sh
$ dbt --debug docs generate -s 'path:models/marts' --vars '{"catalog_db_include": ["db"]}'
...
02:17:41  Building catalog
02:17:41  Acquiring new snowflake connection 'database_no_access.information_schema'
02:17:41  Using snowflake connection "database_no_access.information_schema"
02:17:41  On database_no_access.information_schema: with tables as (
          select
            'PLACEHOLDER' as "table_database",
            'PLACEHOLDER' as "table_schema",
            'PLACEHOLDER' as "table_name",
            'PLACEHOLDER' as "table_type",
            null as "table_comment",
            'PLACEHOLDER' as "table_owner",
            'PLACEHOLDER' as "stats:clustering_key:label",
            null as "stats:clustering_key:value",
            'The key used to cluster this table' as "stats:clustering_key:description",
            false as "stats:clustering_key:include",
            'Row Count' as "stats:row_count:label",
            null as "stats:row_count:value",
            'An approximate count of rows in this table' as "stats:row_count:description",
            null as "stats:row_count:include",
            'Approximate Size' as "stats:bytes:label",
            null as "stats:bytes:value",
            'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
            false as "stats:bytes:include",
            'Last Modified' as "stats:last_modified:label",
            to_varchar(current_timestamp, 'yyyy-mm-dd HH24:MI' || 'UTC') as "stats:last_modified:value",
            'The timestamp for last update/change' as "stats:last_modified:description",
            false as "stats:last_modified:include"
        ),
        columns as (
            select
              'PLACEHOLDER' as "table_database",
              'PLACEHOLDER' as "table_schema",
              'PLACEHOLDER' as "table_name",
              'PLACEHOLDER' as "column_name",
              1 as "column_index",
              'TEXT' as "column_type",
              null as "column_comment"
        )
        select *
          from tables
          join columns using ("table_database", "table_schema", "table_name")
        /* Placeholder query replacing the cataloging of database 'database_no_access' to prevent errors during docs generate with selection. */
        /* Adapted from https://github.com/dbt-labs/dbt-adapters/blob/17c948802a5b4c194354225f76977d78d0fb75c6/dbt-snowflake/src/dbt/include/snowflake/macros/catalog.sql */
/* {"app": "dbt", "dbt_version": "1.11.0b3", "profile_name": "all", "target_name": "sf", "connection_name": "database_no_access.information_schema"} */
02:17:41  Opening a new connection, currently in state init
02:17:44  SQL status: SUCCESS 1 in 3.572 seconds
02:17:44  Re-using an available connection from the pool (formerly database_no_access.information_schema, now db.information_schema)
02:17:44  Using snowflake connection "db.information_schema"
02:17:44  On db.information_schema: with tables as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        case
            when is_dynamic = 'YES' and table_type = 'BASE TABLE' THEN 'DYNAMIC TABLE'
            else table_type
        end as "table_type",
        comment as "table_comment",
        -- note: this is the _role_ that owns the table
        table_owner as "table_owner",
        'Clustering Key' as "stats:clustering_key:label",
        clustering_key as "stats:clustering_key:value",
        'The key used to cluster this table' as "stats:clustering_key:description",
        (clustering_key is not null) as "stats:clustering_key:include",
        'Row Count' as "stats:row_count:label",
        row_count as "stats:row_count:value",
        'An approximate count of rows in this table' as "stats:row_count:description",
        (row_count is not null) as "stats:row_count:include",
        'Approximate Size' as "stats:bytes:label",
        bytes as "stats:bytes:value",
        'Approximate size of the table as reported by Snowflake' as "stats:bytes:description",
        (bytes is not null) as "stats:bytes:include",
        'Last Modified' as "stats:last_modified:label",
        to_varchar(convert_timezone('UTC', last_altered), 'yyyy-mm-dd HH24:MI'||'UTC') as "stats:last_modified:value",
        'The timestamp for last update/change' as "stats:last_modified:description",
        (last_altered is not null and table_type='BASE TABLE') as "stats:last_modified:include"
    from db.INFORMATION_SCHEMA.tables
            where ((
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        ),
        columns as (
            select
        table_catalog as "table_database",
        table_schema as "table_schema",
        table_name as "table_name",
        column_name as "column_name",
        ordinal_position as "column_index",
        data_type as "column_type",
        comment as "column_comment"
    from db.INFORMATION_SCHEMA.columns
            where ((
    "table_schema" ilike 'sch' and upper("table_schema") = upper('sch')
))
        )
        select *
    from tables
    join columns using ("table_database", "table_schema", "table_name")
    order by "column_index"
/* {"app": "dbt", "dbt_version": "1.11.0b3", "profile_name": "all", "target_name": "sf", "connection_name": "db.information_schema"} */
02:17:48  SQL status: SUCCESS 530 in 3.282 seconds
02:17:49  Wrote artifact CatalogArtifact to /Users/jeremy/git/dbt-basic/target/catalog.json
02:17:49  Wrote artifact WritableManifest to /Users/jeremy/git/dbt-basic/target/manifest.json
02:17:49  Wrote artifact SemanticManifest to /Users/jeremy/git/dbt-basic/target/semantic_manifest.json
02:17:49  Catalog written to /Users/jeremy/git/dbt-basic/target/catalog.json
02:17:49  Resource report: {"command_name": "generate", "command_success": true, "command_wall_clock_time": 16.4769, "process_in_blocks": "0", "process_kernel_time": 0.364067, "process_mem_max_rss": "242483200", "process_out_blocks": "0", "process_user_time": 2.464729}
02:17:49  Command `dbt docs generate` succeeded at 15:17:49.727570 after 16.48 seconds
02:17:49  Connection 'generate_catalog' was properly closed.
02:17:49  Connection 'db.information_schema' was left open.
02:17:49  On db.information_schema: Close
```

^ As we can see, when including the var `catalog_db_include` and specifying a list of "allowed databases" (just `db`):

- Database `db` is in the list - therefore run the default catalog query that selects from `db.INFORMATION_SCHEMA`.
- Database `db_no_access` is not in the list - therefore run some simple select query that looks like the actual result set but returns placeholder values.

In this manner, dbt will not exit the invocation with an error. Keep in mind that the caveat of using this method is that you will be responsible for maintaining the `get_catalog` macro as dbt matures and makes future changes to that macro itself.
