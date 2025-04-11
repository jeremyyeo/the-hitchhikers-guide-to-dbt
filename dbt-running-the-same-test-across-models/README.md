---
---

## dbt running the same tests across models

dbt can only run tests, if they are actually declared (i.e. they have to exist and be written to the yaml files) - there aren't too many good shortcuts to that. However, we can use some tools to help us write less code.

### YAML Anchors

https://support.atlassian.com/bitbucket-cloud/docs/yaml-anchors/

```sql
-- models/foo.sql
select 1 id

-- models/bar.sql
select 1 id, 'alice' first_name
```

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
version: "1.0.0"

models:
  my_dbt_project:
    +materialized: table

# models/schema.yml
id_with_test: &id_with_test
  - name: id
    tests:
      - not_null

models:
  - name: foo
    columns:
      - <<: *id_with_test
  - name: bar
    columns:
      - <<: *id_with_test
      - name: first_name
        tests:
          - not_null
```

We define a basic template for a column named `id` that has a test (key it with `&id_with_test`) - and then we just "paste" it to whereever we need it to be (with `*id_with_test`).

```sh
$ dbt build

01:44:52  1 of 5 START sql table model public.bar ........................................ [RUN]
01:44:52  1 of 5 OK created sql table model public.bar ................................... [SELECT 1 in 0.06s]
01:44:52  2 of 5 START sql table model public.foo ........................................ [RUN]
01:44:52  2 of 5 OK created sql table model public.foo ................................... [SELECT 1 in 0.02s]
01:44:52  3 of 5 START test not_null_bar_first_name ...................................... [RUN]
01:44:52  3 of 5 PASS not_null_bar_first_name ............................................ [PASS in 0.03s]
01:44:52  4 of 5 START test not_null_bar_id .............................................. [RUN]

select
      count(*) as failures,
      count(*) != 0 as should_warn,
      count(*) != 0 as should_error
    from (
select
    id as unique_field,
    count(*) as n_records
from "postgres"."public"."bar"
where id is not null
group by id
having count(*) > 1
    ) dbt_internal_test

01:44:52  4 of 5 PASS not_null_bar_id .................................................... [PASS in 0.01s]
01:44:52  5 of 5 START test not_null_foo_id .............................................. [RUN]

select
      count(*) as failures,
      count(*) != 0 as should_warn,
      count(*) != 0 as should_error
    from (
select
    id as unique_field,
    count(*) as n_records

from "postgres"."public"."foo"
where id is not null
group by id
having count(*) > 1
    ) dbt_internal_test

01:44:52  5 of 5 PASS not_null_foo_id .................................................... [PASS in 0.01s]
```

^ We can see here that both model had it's `id` column tested with `not_null` tests. And if we want to also do `unique` tests on the column, then we simply add to the first key:

```yaml
# models/schema.yml
id_with_test: &id_with_test
  - name: id
    tests:
      - not_null
      - unique

models:
  - name: foo
    columns:
      - <<: *id_with_test
  - name: bar
    columns:
      - <<: *id_with_test
      - name: first_name
        tests:
          - not_null
```

```sh
$ dbt build
01:45:08  1 of 7 START sql table model public.bar ........................................ [RUN]
01:45:08  1 of 7 OK created sql table model public.bar ................................... [SELECT 1 in 0.05s]
01:45:08  2 of 7 START sql table model public.foo ........................................ [RUN]
01:45:08  2 of 7 OK created sql table model public.foo ................................... [SELECT 1 in 0.02s]
01:45:08  3 of 7 START test not_null_bar_first_name ...................................... [RUN]
01:45:08  3 of 7 PASS not_null_bar_first_name ............................................ [PASS in 0.02s]
01:45:08  4 of 7 START test not_null_bar_id .............................................. [RUN]
01:45:08  4 of 7 PASS not_null_bar_id .................................................... [PASS in 0.01s]
01:45:08  5 of 7 START test unique_bar_id ................................................ [RUN]
01:45:08  5 of 7 PASS unique_bar_id ...................................................... [PASS in 0.02s]
01:45:08  6 of 7 START test not_null_foo_id .............................................. [RUN]
01:45:08  6 of 7 PASS not_null_foo_id .................................................... [PASS in 0.01s]
01:45:08  7 of 7 START test unique_foo_id ................................................ [RUN]
01:45:08  7 of 7 PASS unique_foo_id ...................................................... [PASS in 0.01s]
```

### Singular data test

https://docs.getdbt.com/docs/build/data-tests#singular-data-tests

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
version: "1.0.0"

models:
  my_dbt_project:
    +materialized: table
```

```sql
-- models/foo.sql
select 1 id

-- models/bar.sql
select 1 id, 'alice' first_name

-- tests/assert_all_models_have_not_null_id.sql
{%- if execute -%}
    {%- set all_models = [] -%}
    {%- for n in graph.nodes.values() | selectattr("resource_type", "equalto", "model") -%}
      {%- do all_models.append(n.relation_name) -%}
    {%- endfor -%}
    {% for m in all_models %}
        select 1 as n_record from {{ m }} where id is null
        {% if not loop.last -%}union all{% endif -%}
    {% endfor %}
{%- endif -%}
```

```sh
$ dbt build
02:06:24  1 of 3 START sql table model public.bar ........................................ [RUN]
02:06:24  1 of 3 OK created sql table model public.bar ................................... [SELECT 1 in 0.06s]
02:06:24  2 of 3 START sql table model public.foo ........................................ [RUN]
02:06:24  2 of 3 OK created sql table model public.foo ................................... [SELECT 1 in 0.02s]
02:06:24  3 of 3 START test assert_all_models_have_not_null_id ........................... [RUN]

select
      count(*) as failures,
      count(*) != 0 as should_warn,
      count(*) != 0 as should_error
    from (
      
        select 1 as n_record from "postgres"."public"."bar" where id is null
        union all
        select 1 as n_record from "postgres"."public"."foo" where id is null
      
    ) dbt_internal_test

02:06:24  3 of 3 PASS assert_all_models_have_not_null_id ................................. [PASS in 0.05s]
```

You'll notice that unlike previously, we're doing this all at once go - instead of one model at a time - so this may not be what you want - for example if we added a third model that would fail the test:

```sql
-- models/baz.sql
select null id
```

```sh
$ dbt build
02:12:02  1 of 4 START sql table model public.bar ........................................ [RUN]
02:12:02  1 of 4 OK created sql table model public.bar ................................... [SELECT 1 in 0.06s]
02:12:02  2 of 4 START sql table model public.baz ........................................ [RUN]
02:12:02  2 of 4 OK created sql table model public.baz ................................... [SELECT 1 in 0.02s]
02:12:02  3 of 4 START sql table model public.foo ........................................ [RUN]
02:12:02  3 of 4 OK created sql table model public.foo ................................... [SELECT 1 in 0.05s]
02:12:02  4 of 4 START test assert_all_models_have_not_null_id ........................... [RUN]

select
      count(*) as failures,
      count(*) != 0 as should_warn,
      count(*) != 0 as should_error
    from (
      
        select 1 as n_record from "postgres"."public"."bar" where id is null
        union all
        select 1 as n_record from "postgres"."public"."foo" where id is null
        union all
        select 1 as n_record from "postgres"."public"."baz" where id is null
        
      
    ) dbt_internal_test

02:12:02  4 of 4 FAIL 1 assert_all_models_have_not_null_id ............................... [FAIL 1 in 0.02s]
```

There's a couple of downsides to doing this - because we are not using `{{ ref() }}` in our data tests - there is not direct linking (DAG) between the data test and the model itself - so if we do a build or test on a particular model, the test itself wont run:

```sh
$ dbt build -s foo
02:23:42  1 of 1 START sql table model public.foo ........................................ [RUN]
02:23:42  1 of 1 OK created sql table model public.foo ................................... [SELECT 1 in 0.05s]

$ dbt test -s foo
02:23:57  Nothing to do. Try checking your model configs and model specification args
```

Since the singular data test is decoupled from the models, dbt will not guarantee that the test will 100% run only after the models are being built - unlike the case above where we declared in the schema yaml file that each model had a column named `id` with a `not_null` test.

### Hooks

We can also do this "test" via hooks:


```sql
-- models/foo.sql
select 1 id

-- models/bar.sql
select null id

-- macros/check_for_nulls.sql
{% macro check_for_nulls(this) %}
    {% if execute %}
        {% set query %}
            select * from {{ this }} where id is null
        {% endset %}
        {% set res = run_query(query).columns[0].values() | length %}
        {% if res > 0 %}
            {% do exceptions.raise_compiler_error('NULL ids found in ' ~ this) %}
        {% else %}
            {% do log('No NULL ids.') %}
        {% endif %}
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
    +post_hook: "{{ check_for_nulls(this) }}"
```

```sh
$ dbt build

02:35:39  1 of 2 START sql table model public.bar ........................................ [RUN]
02:35:39  1 of 2 ERROR creating sql table model public.bar ............................... [ERROR in 0.05s]
02:35:39  2 of 2 START sql table model public.foo ........................................ [RUN]
02:35:39  2 of 2 OK created sql table model public.foo ................................... [SELECT 1 in 0.03s]
02:35:39  
02:35:39  Finished running 2 table models in 0 hours 0 minutes and 0.21 seconds (0.21s).
02:35:39  
02:35:39  Completed with 1 error, 0 partial successes, and 0 warnings:
02:35:39  
02:35:39  Failure in model bar (models/bar.sql)
02:35:39    Compilation Error in model bar (models/bar.sql)
  NULL ids found in "postgres"."public"."bar"
  
  > in macro check_for_nulls (macros/check_for_nulls.sql)
  > called by macro run_hooks (macros/materializations/hooks.sql)
  > called by macro materialization_table_default (macros/materializations/models/table.sql)
  > called by model bar (models/bar.sql)
02:35:39  
02:35:39    compiled code at target/compiled/my_dbt_project/models/bar.sql
02:35:39  
02:35:39  Done. PASS=1 WARN=0 ERROR=1 SKIP=0 NO-OP=0 TOTAL=2
```

Of course, this isn't a "test" per se so `dbt test` will not do anything:

```sh
$ dbt test
02:37:17  Nothing to do. Try checking your model configs and model specification args
```
