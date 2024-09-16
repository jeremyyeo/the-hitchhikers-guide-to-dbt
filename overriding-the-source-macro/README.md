---
---

## Overriding the source macro

Here's quick Snowflake example on overriding the source macro so that we can add some additional keywords (such as table sampling) to the source. 

P.s. it's probably a good idea not to override how the `source()` builtin works and do this table sampling within the model itself - you may run into unknown issues if you choose to go down this path.

First let's create 2 source tables with 10 rows each:

```sql
create or replace table db.dbt_jyeo.customers as 
select value as c from table(flatten(input => array_generate_range(1, 10)))
;

create or replace table db.dbt_jyeo.orders as 
select value as c from table(flatten(input => array_generate_range(1, 10)))
;
```

Then we can make use of our sources in our dbt project:

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
config-version: 2
version: "1.0.0"

models:
  my_dbt_project:
    +materialized: table

# models/sources.yml
sources:
  - name: dbt_jyeo
    tables:
      - name: customers
        meta: 
          sample: 5
      - name: orders
```

```sql
-- models/foo.sql
select * from {{ source('dbt_jyeo', 'customers') }}

-- models/bar.sql
select * from {{ source('dbt_jyeo', 'orders') }}
```

And then the `source` macro override:

```sql
-- macros/source.sql
{% macro source(source_name, model_name) %}
    {% set ns = namespace(source=false) %}
    {% if execute %}
    {% for node in graph.sources.values() %}
        {% if node.source_name == source_name and node.name == model_name %}
            {% set ns.source = node %}
        {% endif %}
    {% endfor %}
    {% endif %}
    {% if ns.source.meta and ns.source.meta.sample %}
        {% set fqn -%}
            (select * from {{ builtins.source(source_name, model_name)}} sample ({{ ns.source.meta.sample }} rows))
        {%- endset %}
    {% else %}
        {% set fqn = builtins.source(source_name, model_name) %}
    {% endif %}
    {% do return(fqn) %}
{% endmacro %}
```

Let's build our models:

```sh
$ dbt --debug build
...
03:20:29  On model.my_dbt_project.bar: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.bar"} */
create or replace transient table db.schema.bar
         as
        (select * from db.dbt_jyeo.orders
        );

03:20:32  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.5", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.foo"} */
create or replace transient table db.schema.foo
         as
        (select * from (select * from db.dbt_jyeo.customers sample (5 rows))
        );

$ dbt show -s foo
03:20:58  Running with dbt=1.8.5
03:20:59  Registered adapter: snowflake=1.8.3
03:20:59  Found 2 models, 2 sources, 449 macros
03:20:59  
03:21:01  Concurrency: 1 threads (target='sf')
03:21:01  
03:21:02  Previewing node 'foo':
| C |
| - |
| 1 |
| 2 |
| 9 |
| 4 |
| 7 |
```

As we can see from the DDL and the model output, we successfully added sampling to the source table as it is being used by the `foo` model.

Again, it's probably best not to do this by overrding the `source()` builtin but to add your sampling to the model itself as overriding `source()` may lead to issues down the line.

```sql
-- models/foo.sql
select * from ({{ source('dbt_jyeo', 'customers') }} sample (5 rows))
```
