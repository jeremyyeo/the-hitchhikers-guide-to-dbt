---
---

## Configurations in files

Examples of how to store arbitray configs in separates files.

### In macros

The most straightforward way is to simply configs in variables:

```yaml
# dbt_project.yml
...
vars:
    business_days: ["mon", "tue", "wed", "thu", "fri"]
```

However, some folks want a separate file for this as they have potentially dozens or hundreds of "configs" which makes storing them in vars in the `dbt_project.yml` file become quite tedious over time.

We can make a macro that is "vars" like:

```sql
-- macros/cvar.sql
{% macro cvar(var_name) -%}
    {#/* Add new keys to all_project_vars as required. */#}
    {%- 
        set all_project_vars = {
            "business_days": ["mon", "tue", "wed", "thu", "fri"]
        }
    -%}
    {{ return(all_project_vars[var_name]) }}
{%- endmacro %}
```

And then use it in our project:

```sql
-- models/foo.sql
{%- for bd in cvar("business_days") -%}
select '{{ bd }}' as business_days
{% if not loop.last -%}union all{% endif %}
{% endfor -%}
```

```sh
$ dbt compile -s foo
23:34:10  Running with dbt=1.8.4
23:34:10  Registered adapter: postgres=1.8.2
23:34:10  Found 1 model, 1 exposure, 532 macros
23:34:10  
23:34:10  Concurrency: 4 threads (target='pg')
23:34:10  
23:34:10  Compiled node 'foo' is:
select 'mon' as business_days
union all
select 'tue' as business_days
union all
select 'wed' as business_days
union all
select 'thu' as business_days
union all
select 'fri' as business_days
```

Simply keep adding new keys to our macro as required.

https://gist.github.com/jeremyyeo/06d552ee8facc8100416655ebc25d9b9

### In yml files

dbt cannot actually read arbitrary yml files... let's say you have some random yml file like:

```yaml
# models/my_file.yml
business_days: ["mon", "tue", "wed", "thu", "fri"]
are_you_the_balrog: yes
```

dbt doesn't actually read/parse that file into anything meaningful in your dbt project. This is because dbt will only parse top level keys of yml files that *mean something* to it - such as:

```yaml
# models/my_file.yml
models:
 ...
sources:
 ...
```

What we can do is then store our configs in a `meta` config under something like an exposure that we do not use:

```yaml
# macros/__configs.yml
# Not a real exposure - add new configs to the `meta` key.
exposures:
  - name: __configs__
    type: application
    owner: 
      name: Config
    meta:
    # Add any new config we want to use here.
      business_days: ["mon", "tue", "wed", "thu", "fri"]
      are_you_the_balrog: yes
```

Then add a macro to fetch the configs:

```sql
-- macros/__get_configs.sql
{% macro get_configs(config_name) %}
    {% if execute %}
        {% set config = graph.exposures.values() | selectattr("name", "equalto", "__configs__") | list %}
        {{ return(config[0].meta[config_name]) }}
    {% else %}
        {{ return('') }}
    {% endif %}
{% endmacro %}
```

We can then use the `get_configs()` macro in our project:

```sql
-- models/foo.sql
{%- for bd in get_configs("business_days") -%}
select '{{ bd }}' as business_days
{% if not loop.last -%}union all{% endif %}
{% endfor -%}
```

```sh
$ dbt compile -s foo
23:22:35  Running with dbt=1.8.4
23:22:35  Registered adapter: postgres=1.8.2
23:22:35  Found 1 model, 1 exposure, 531 macros
23:22:35  
23:22:35  Concurrency: 4 threads (target='pg')
23:22:35  
23:22:35  Compiled node 'foo' is:
select 'mon' as business_days
union all
select 'tue' as business_days
union all
select 'wed' as business_days
union all
select 'thu' as business_days
union all
select 'fri' as business_days
```

In another macro:

```sql
-- macros/ops.sql
{% macro can_i_pass() %}
    {% if get_configs("are_you_the_balrog") %}
        {% do print("You shall not pass.") %}
    {% else %}
        {% do print("Sure.") %}
    {% endif %}
{% endmacro %}
```

```sh
$ dbt run-operation can_i_pass
23:25:05  Running with dbt=1.8.4
23:25:05  Registered adapter: postgres=1.8.2
23:25:05  Found 1 model, 1 exposure, 531 macros
You shall not pass.
```

And if we change our yml:

```yaml
# macros/__configs.yml
# Not a real exposure - add new configs to the `meta` key.
exposures:
  - name: __configs__
    type: application
    owner: 
      name: Config
    meta:
    # Add any new config we want to use here.
      business_days: ["mon", "tue", "wed", "thu", "fri"]
      are_you_the_balrog: no # Make Gandalf let us through.
```

```sh
$ dbt run-operation can_i_pass
23:26:00  Running with dbt=1.8.4
23:26:00  Registered adapter: postgres=1.8.2
23:26:00  Found 1 model, 1 exposure, 531 macros
Sure.
```



