---
---

## Checking dwh parameters

Every so often, one may need to double check that the datawarehouse parameters set are actually of the right values. Here's a Snowflake example on how to log those out so that we know for sure the params are the right values we think they are:

```sql
-- macros/check_params.sql
{% macro check_params() %}
    {% if execute %}
        {% for obj in ['account', 'user', 'warehouse'] %}
            {% set query %}
                show parameters in {{ obj }};
                select "key", "value" from table(result_scan(last_query_id())) where "key" ilike any ('TIMEZONE', '%TIMEOUT%');
            {% endset %}
            {% set res = run_query(query) %}
            {% do log('=' * 80) %}
            {% for k,v in res %}
                {% do log(obj ~ ' / ' ~ k ~ ': ' ~ v) %}
            {% endfor %}
            {% do log('=' * 80) %}
        {% endfor %}
    {% endif %}
{% endmacro %}
```

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
version: "1.0.0"

models:
  my_dbt_project:
    +materialized: table

on-run-start: "{{ check_params() }}"
```

```sh
$ dbt --debug run
...
05:05:50  Concurrency: 1 threads (target='sf')
...
05:05:53  Using snowflake connection "master"
05:05:53  On master: /* {"app": "dbt", "dbt_version": "1.9.4", "profile_name": "all", "target_name": "sf", "connection_name": "master"} */
show parameters in account;
05:05:53  Opening a new connection, currently in state init
05:05:54  SQL status: SUCCESS 202 in 1.703 seconds
05:05:54  Using snowflake connection "master"
05:05:54  On master: /* {"app": "dbt", "dbt_version": "1.9.4", "profile_name": "all", "target_name": "sf", "connection_name": "master"} */
select "key", "value" from table(result_scan(last_query_id())) where "key" ilike any ('TIMEZONE', '%TIMEOUT%');
05:05:55  SQL status: SUCCESS 7 in 1.079 seconds
05:05:55  ================================================================================
05:05:55  account / HYBRID_TABLE_LOCK_TIMEOUT: 3600
05:05:55  account / LOCK_TIMEOUT: 43200
05:05:55  account / SNOWPARK_REQUEST_TIMEOUT_IN_SECONDS: 86400
05:05:55  account / STATEMENT_QUEUED_TIMEOUT_IN_SECONDS: 0
05:05:55  account / STATEMENT_TIMEOUT_IN_SECONDS: 172800
05:05:55  account / TIMEZONE: America/Los_Angeles
05:05:55  account / USER_TASK_TIMEOUT_MS: 3600000
05:05:55  ================================================================================
05:05:55  Using snowflake connection "master"
05:05:55  On master: /* {"app": "dbt", "dbt_version": "1.9.4", "profile_name": "all", "target_name": "sf", "connection_name": "master"} */
show parameters in user;
05:05:56  SQL status: SUCCESS 142 in 0.323 seconds
05:05:56  Using snowflake connection "master"
05:05:56  On master: /* {"app": "dbt", "dbt_version": "1.9.4", "profile_name": "all", "target_name": "sf", "connection_name": "master"} */
select "key", "value" from table(result_scan(last_query_id())) where "key" ilike any ('TIMEZONE', '%TIMEOUT%');
05:05:57  SQL status: SUCCESS 6 in 0.743 seconds
05:05:57  ================================================================================
05:05:57  user / HYBRID_TABLE_LOCK_TIMEOUT: 3600
05:05:57  user / LOCK_TIMEOUT: 43200
05:05:57  user / SNOWPARK_REQUEST_TIMEOUT_IN_SECONDS: 86400
05:05:57  user / STATEMENT_QUEUED_TIMEOUT_IN_SECONDS: 0
05:05:57  user / STATEMENT_TIMEOUT_IN_SECONDS: 172800
05:05:57  user / TIMEZONE: America/Los_Angeles
05:05:57  ================================================================================
05:05:57  Using snowflake connection "master"
05:05:57  On master: /* {"app": "dbt", "dbt_version": "1.9.4", "profile_name": "all", "target_name": "sf", "connection_name": "master"} */
show parameters in warehouse;
05:05:57  SQL status: SUCCESS 3 in 0.347 seconds
05:05:57  Using snowflake connection "master"
05:05:57  On master: /* {"app": "dbt", "dbt_version": "1.9.4", "profile_name": "all", "target_name": "sf", "connection_name": "master"} */
select "key", "value" from table(result_scan(last_query_id())) where "key" ilike any ('TIMEZONE', '%TIMEOUT%');
05:05:58  SQL status: SUCCESS 2 in 0.739 seconds
05:05:58  ================================================================================
05:05:58  warehouse / STATEMENT_QUEUED_TIMEOUT_IN_SECONDS: 0
05:05:58  warehouse / STATEMENT_TIMEOUT_IN_SECONDS: 172800
05:05:58  ================================================================================
05:05:58  Writing injected SQL for node "operation.my_dbt_project.my_dbt_project-on-run-start-0"
05:05:58  1 of 1 START hook: my_dbt_project.on-run-start.0 ............................... [RUN]
05:05:58  1 of 1 OK hook: my_dbt_project.on-run-start.0 .................................. [OK in 4.97s]
...
```