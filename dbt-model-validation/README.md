---
---

## The dbt model validation

### Model name validation

Let's assume we have a dbt project with a models folder structure as follows:

```
models/
├── dimensions/
│   └── dim_users.sql
├── staging/
│   └── stg_users.sql
└── fct_users.sql
```

And we want to validate that each model in the `dimensions/` folder has a name that starts with `dim_` and each model in the `staging/` folder has a name that starts with `stg_`. We also do not care too much about what other models are named if they are outside of those 2 folders.

#### Using a pre-hook macro

We can accomplish this by running a hook just before a model is executed (`run`).

1. Add a macro that we will use on our model pre-hook:

```jinja
-- macros/validate_model.sql

{% macro validate_model(this, prefix) %}

  {% if execute %}

    {% for node in graph.nodes.values() %}
      {% if node.name == this.identifier %}

        {% if node.name.startswith(prefix) %}

          {% set message = "model in path <" ~ node.original_file_path ~ "> contains the right prefix of <" ~ prefix ~ ">" %}
          {% do log(message, True) %}

        {% else %}

          {% set message = "model in path <" ~ node.original_file_path ~ "> DOES NOT CONTAIN the right prefix of <" ~ prefix ~ ">" %}
          {% do exceptions.raise_compiler_error(message) %}

        {% endif %}

      {% endif %}
    {% endfor %}

  {% endif %}

{% endmacro %}
```

The macro takes in a relation with the [`this` variable](https://docs.getdbt.com/reference/dbt-jinja-functions/this) that refers to the current model and a `prefix` which we will set to a string representing the prefix we want our models to have.

The macro also uses the [`graph` context variable](https://docs.getdbt.com/reference/dbt-jinja-functions/graph) which contains a dictionary of all our nodes / models and it's attributes (such as the complete file path to the model, the type of materialization, etc).

2. Let's use our macro in a pre-hook.

```yml
# dbt_project.yml

name: "my_project"

---
models:
  my_project:
    +materialized: table
    dimensions:
      +pre-hook: "{{ validate_model(this, 'dim_') }}"
    staging:
      +pre-hook: "{{ validate_model(this, 'stg_') }}"
```

3. Let's give our project a run.

```
$ dbt run

01:52:59  Running with dbt=1.0.2
01:53:00  Found 3 models, 0 tests, 0 snapshots, 0 analyses, 180 macros, 0 operations, 0 seed files, 0 sources, 0 exposures, 0 metrics
01:53:00
01:53:05  Concurrency: 1 threads (target='dev')
01:53:05
01:53:05  1 of 3 START table model dbt_jyeo.dim_users..................................... [RUN]
01:53:05  model in path <models/dimensions/dim_users.sql> contains the right prefix of <dim_>
01:53:09  1 of 3 OK created table model dbt_jyeo.dim_users................................ [SUCCESS 1 in 3.38s]
01:53:09  2 of 3 START table model dbt_jyeo.fct_users..................................... [RUN]
01:53:12  2 of 3 OK created table model dbt_jyeo.fct_users................................ [SUCCESS 1 in 3.37s]
01:53:12  3 of 3 START table model dbt_jyeo.stg_users..................................... [RUN]
01:53:12  model in path <models/staging/stg_users.sql> contains the right prefix of <stg_>
01:53:15  3 of 3 OK created table model dbt_jyeo.stg_users................................ [SUCCESS 1 in 3.25s]
01:53:15
01:53:15  Finished running 3 table models in 15.70s.
01:53:15
01:53:15  Completed successfully
01:53:15
01:53:15  Done. PASS=3 WARN=0 ERROR=0 SKIP=0 TOTAL=3
```

Everything worked as expected. Let's now try to trigger an error.

4. Add a `customers.sql` model to the `dimensions/` folder with any arbitrary content in it (all the models in this example just have `select 1 as id` in it). Our models folder structure should now look like:

```
models/
├── dimensions/
│   ├── customers.sql
│   └── dim_users.sql
├── staging/
│   └── stg_users.sql
└── fct_users.sql
```

5. Let's give our project another run.

```
$ dbt run

02:03:20  Running with dbt=1.0.2
02:03:21  Found 4 models, 0 tests, 0 snapshots, 0 analyses, 180 macros, 0 operations, 0 seed files, 0 sources, 0 exposures, 0 metrics
02:03:21
02:03:27  Concurrency: 1 threads (target='dev')
02:03:27
02:03:27  1 of 4 START table model dbt_jyeo.customers..................................... [RUN]
02:03:27  1 of 4 ERROR creating table model dbt_jyeo.customers............................ [ERROR in 0.04s]
02:03:27  2 of 4 START table model dbt_jyeo.dim_users..................................... [RUN]
02:03:27  model in path <models/dimensions/dim_users.sql> contains the right prefix of <dim_>
02:03:30  2 of 4 OK created table model dbt_jyeo.dim_users................................ [SUCCESS 1 in 3.68s]
02:03:30  3 of 4 START table model dbt_jyeo.fct_users..................................... [RUN]
02:03:34  3 of 4 OK created table model dbt_jyeo.fct_users................................ [SUCCESS 1 in 3.60s]
02:03:34  4 of 4 START table model dbt_jyeo.stg_users..................................... [RUN]
02:03:34  model in path <models/staging/stg_users.sql> contains the right prefix of <stg_>
02:03:37  4 of 4 OK created table model dbt_jyeo.stg_users................................ [SUCCESS 1 in 3.41s]
02:03:37
02:03:37  Finished running 4 table models in 16.66s.
02:03:37
02:03:37  Completed with 1 error and 0 warnings:
02:03:37
02:03:37  Compilation Error in model customers (models/dimensions/customers.sql)
02:03:37    model in path <models/dimensions/customers.sql> DOES NOT CONTAIN the right prefix of <dim_>
02:03:37
02:03:37    > in macro validate_model (macros/validate_model.sql)
02:03:37    > called by macro run_hooks (macros/materializations/hooks.sql)
02:03:37    > called by macro materialization_table_snowflake (macros/materializations/table.sql)
02:03:37    > called by model customers (models/dimensions/customers.sql)
02:03:37
02:03:37  Done. PASS=3 WARN=0 ERROR=1 SKIP=0 TOTAL=4
```

We now see that the `customers.sql` model raised an error (expectedly) because it did not contain the right prefix.

#### Using a run-operation macro

Let's now see how we can validate our models using a `run-operation` - this can come in handy if you want to validate every model first before you actually proceed to `dbt run`.

1. Add a macro that we will use with our run-operation:

```jinja
-- macros/validate_all_models.sql

{% macro validate_all_model() %}


{% endmacro %}
```

Left up to the reader.

### Models contain certain columns

Validate that models must contain certain columns.

```sql
-- macros/validate_model_columns.sql
{% macro validate_model_columns(this, includes) %}
  {% if execute %}
    {% set cols = adapter.get_columns_in_relation(this) %}
    {% set cols_in_rel = cols | map(attribute="column") | list | lower %}
    {% set includes = includes | list %}
    {% set diff = includes | reject('in', cols_in_rel) | list %}
    {% if diff | length > 0 %}
      {% do exceptions.raise_compiler_error('Model ' ~ this ~ ' does not include required columns ' ~ diff) %}
    {% endif %}
  {% endif %}
{% endmacro %}

-- models/foo.sql
select 1 as id, current_date() as updated_at, 'alice' as first_name

-- models/bar.sql
select 'alice' as first_name

-- models/baz.sql
select 1 id, 'alice' as first_name
```

```yaml
# dbt_project.yml
name: analytics
profile: sf
version: "1.0.0"

models:
  analytics:
    +materialized: table
    +post-hook: "{{ validate_model_columns(this, includes=['id', 'updated_at']) }}"
```

```sh
$ dbt run

23:23:07  Running with dbt=1.10.0-rc2
23:23:07  Registered adapter: snowflake=1.9.4
23:23:07  Found 3 models, 589 macros
23:23:07
23:23:07  Concurrency: 1 threads (target='ci')
23:23:07
23:23:09  1 of 3 START sql table model sch.bar ........................................... [RUN]
23:23:10  1 of 3 ERROR creating sql table model sch.bar .................................. [ERROR in 1.08s]
23:23:10  2 of 3 START sql table model sch.baz ........................................... [RUN]
23:23:12  2 of 3 ERROR creating sql table model sch.baz .................................. [ERROR in 1.21s]
23:23:12  3 of 3 START sql table model sch.foo ........................................... [RUN]
23:23:13  3 of 3 OK created sql table model sch.foo ...................................... [SUCCESS 1 in 1.39s]
23:23:13
23:23:13  Finished running 3 table models in 0 hours 0 minutes and 6.00 seconds (6.00s).
23:23:14
23:23:14  Completed with 2 errors, 0 partial successes, and 0 warnings:
23:23:14
23:23:14  Failure in model bar (models/bar.sql)
23:23:14    Compilation Error in model bar (models/bar.sql)
  Model db.sch.bar does not include required columns ['id', 'updated_at']

  > in macro validate_model_columns (macros/validate_model_columns.sql)
  > called by macro run_hooks (macros/materializations/hooks.sql)
  > called by macro materialization_table_snowflake (macros/materializations/table.sql)
  > called by model bar (models/bar.sql)
23:23:14
23:23:14    compiled code at target/compiled/analytics/models/bar.sql
23:23:14
23:23:14  Failure in model baz (models/baz.sql)
23:23:14    Compilation Error in model baz (models/baz.sql)
  Model db.sch.baz does not include required columns ['updated_at']

  > in macro validate_model_columns (macros/validate_model_columns.sql)
  > called by macro run_hooks (macros/materializations/hooks.sql)
  > called by macro materialization_table_snowflake (macros/materializations/table.sql)
  > called by model baz (models/baz.sql)
23:23:14
23:23:14    compiled code at target/compiled/analytics/models/baz.sql
23:23:14
23:23:14  Done. PASS=1 WARN=0 ERROR=2 SKIP=0 NO-OP=0 TOTAL=3
```
