---
---

# dbt Macro context

> This appear to be an undocumented feature and dbt-fusion does not yet have support for this pattern (https://github.com/dbt-labs/dbt-fusion/issues/787).

It's possible to call a macro with `context['name_of_macro'](...args)`.

```sql
-- macros/multiplier.sql
{% macro double(i) %}
    {{ return(i * 2) }}
{% endmacro %}

{% macro triple(i) %}
    {{ return(i * 3) }}
{% endmacro %}

-- models/foo.sql
select {{ context['double'](10) }} as d, {{ context['triple'](10) }} as t
```

```sh
$ dbt --debug run
...
22:03:17  On model.analytics.foo: create or replace transient table db.ci.foo
    as (select 20 as d, 30 as t
    )
...
```

With this, we can dynamically (based on a query result) determine which macro to use.

```sql
-- macros/multiplier.sql
{% macro multiplier(i) %}
    {% if execute %}
        {% set res = run_query('select which_macro from db.sch.which_macro') %}
        {% set m = res.columns[0].values()[0] %}
        {{ return(context[m](i)) }}
    {% endif %}
{% endmacro %}

{% macro double(i) %}
    {{ return(i * 2) }}
{% endmacro %}

{% macro triple(i) %}
    {{ return(i * 3) }}
{% endmacro %}

-- models/foo.sql
select {{ multiplier(10) }} as c
```

If our "control table" `db.sch.which_macro` looks like:

```sql
-- ran directly on the dwh.
create or replace table db.sch.which_macro as (select 'double' as which_macro);
```

Then we can see `foo` being built as:

```sh
$ dbt --debug run
...
22:06:07  On model.analytics.foo: create or replace transient table db.ci.foo
    as (select 20 as c
    )
```

And if the control table instead had:

```sql
-- ran directly on the dwh.
create or replace table db.sch.which_macro as (select 'triple' as which_macro);
```

We then see `foo` being built as:

```sh
$ dbt --debug run
...
22:07:19  On model.analytics.foo: create or replace transient table db.ci.foo
    as (select 30 as c
    )
```

^ There was no need to modify any dbt project code - the determination of which macro to use was based on the value being returned after querying our "control table".
