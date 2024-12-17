---
---

## Using the dbt graph context variable

The dbt `graph` context variable contains the information about the nodes in our project which we can use to do fun things with.

https://docs.getdbt.com/reference/dbt-jinja-functions/graph

### Checking for models that have no tests

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
config-version: 2
version: "1.0.0"

models:
  my_dbt_project:
    +materialized: table

# models/schema.yml
models:
  - name: foo
    columns:
      - name: id
        tests:
          - unique
```

```sql
-- models/foo.sql
select 1 id

-- models/bar.sql
select 1 id

-- macros/models_without_tests.sql
{% macro models_without_tests() %}
    {% set all_models = graph.nodes.values() | selectattr("resource_type", "equalto", "model") | map(attribute="unique_id") | list %}
    {% set all_tests = graph.nodes.values() | selectattr("resource_type", "equalto", "test") | map(attribute="attached_node") | list %}
    {% for model in all_models %}
        {% if model not in all_tests %}
            {% do print('Model ' ~ model ~ ' has no tests defined.') %}
        {% endif %}
    {% endfor %}
{% endmacro %}
```

```sh
$ dbt run-operation models_without_tests
03:29:04  Running with dbt=1.9.0-rc2
03:29:05  Registered adapter: snowflake=1.9.0-rc1
03:29:05  Found 2 models, 1 test, 469 macros
Model model.my_dbt_project.bar has no tests defined.
```

^ As we can see, we didn't have any test on model `bar` - and that's what was printed out by our run-operation.

### Other examples

https://gist.github.com/jeremyyeo/83adf1f412e5e497baef60e5ada35bf8
