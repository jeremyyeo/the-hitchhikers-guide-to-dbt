---
---

## dbt grants

https://docs.getdbt.com/reference/resource-configs/grants

### Basics (and Snowflake copy_grants)

https://www.youtube.com/watch?v=qtKWz5NsWo0

### Conditionally applying grants (e.g. based on `target.name`)

```sql
-- models/foo.sql
{{
  config(
    materialized='table',
    grants=(
        {'select': ['tester']} if target.name == 'prod' else {}
    )
  )
}}

select 1 id
```

When the target is named `prod`, then the grants config resolves to `{'select': ['tester']}` and therefore, a `grant select ... to tester` is executed after the model is built:

```sh
$ dbt --debug run -t prod
...
01:27:02  1 of 1 START sql table model prod.foo .......................................... [RUN]

01:27:02  On model.analytics.foo: create or replace transient table db.prod.foo
    as (
select 1 id
    )
/* {"app": "dbt", "dbt_version": "1.10.4", "profile_name": "sf", "target_name": "prod", "node_id": "model.analytics.foo"} */;
01:27:03  SQL status: SUCCESS 1 in 1.079 seconds

01:27:03  On model.analytics.foo: grant select on db.prod.foo to tester
/* {"app": "dbt", "dbt_version": "1.10.4", "profile_name": "sf", "target_name": "prod", "node_id": "model.analytics.foo"} */;
01:27:04  SQL status: SUCCESS 1 in 0.399 seconds

01:27:04  1 of 1 OK created sql table model prod.foo ..................................... [SUCCESS 1 in 1.56s]
```

When the target is not named `prod`, then the grants config resolves to an empty dict `{}` and therefore no grant sql statements are executed:

```sh
$ dbt --debug run -t ci
...
01:27:51  1 of 1 START sql table model sch.foo ........................................... [RUN]

01:27:51  On model.analytics.foo: create or replace transient table db.ci.foo
    as (
select 1 id
    )
/* {"app": "dbt", "dbt_version": "1.10.4", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
01:27:52  SQL status: SUCCESS 1 in 1.098 seconds

01:27:52  1 of 1 OK created sql table model sch.foo ...................................... [SUCCESS 1 in 1.15s]
```

It's not possible to use this pattern in the `dbt_project.yml` file so you'll have to add it to each model's config block. We can use a macro to keep things DRY:

```sql
-- macros/grant_condish.sql
{% macro grant_condish() -%}
    {{ return({'select': ['tester']} if target.name == 'prod' else {}) }}
{%- endmacro %}
```

```sql
-- models/foo.sql
{{ config(materialized='table', grants=grant_condish()) }}
select 1 id

-- models/bar.sql
{{ config(materialized='table', grants=grant_condish()) }}
select 1 id
```

Notice that the way we're calling the macro is not typical of how we'd normally do that (e.g. jinja in string like `some_key = "{{ my_macro() }}"`) which may have implications on `state:modified` - that will be left to the reader to test/experiment.
