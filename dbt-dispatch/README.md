---
---

## dbt dispatch

https://docs.getdbt.com/reference/dbt-jinja-functions/dispatch

### Applying global macros (e.g. `generate_schema_name`) from packages from the root dbt project

https://docs.getdbt.com/reference/dbt-jinja-functions/dispatch#overriding-global-macros

Assuming we have a shared "utilitly" dbt packages that we want to share with all our other dbt projects:

```yaml
# shared_pkg/dbt_project.yml
name: shared_pkg
```

```sql
-- shared_pkg/macros/generate_schema_name.sql
{% macro default__generate_schema_name(custom_schema_name, node) -%}
    {{ 'from_shared_dbt_pkg' }}
{%- endmacro %}
```

^ Here we want to dictate the "generate schema name" logic that would apply to our root dbt project from the `shared_pkg` package, we would install the package into our dbt project:

```yaml
# analytics/dbt_project.yml
name: analytics
profile: sf
version: "1.0.0"

models:
  analytics:
    +materialized: table

# analytics/packages.yml
packages:
  - local: ../shared_pkg # install the package into your dbt project using the right method - here I'm just installing it locally.
```

```sql
-- analytics/models/foo.sql
select '{{ target.schema }}' as c
```

```sh
$ dbt run
07:00:32  1 of 1 START sql table model ci.foo ............................................ [RUN]
07:00:32  On model.analytics.foo: create or replace transient table db.ci.foo
    as (select 'ci' as c
    )
/* {"app": "dbt", "dbt_version": "1.11.5", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
07:00:33  1 of 1 OK created sql table model ci.foo ....................................... [SUCCESS 1 in 1.30s]
```

Here we see that it doesn't quite work - the schema of the model is `ci` which is the schema set in the connection (`target.schema`) - this means the generate schema name logic from our package is not being applied. This is because we have yet to use the dispatch mechanism... let's go ahead and do that by modifying the `dbt_project.yml` file:

```yaml
# analytics/dbt_project.yml
name: analytics
profile: sf
version: "1.0.0"

models:
  analytics:
    +materialized: table

dispatch:
  - macro_namespace: dbt
    search_order: ["analytics", "shared_pkg", "dbt"]
# the dbt package that we're installing must be ahead of "dbt" in the search order.
```

Once we do that...

```sh
$ dbt run
07:03:26  1 of 1 START sql table model gsn_from_pkg.foo .................................. [RUN]
07:03:26  On model.analytics.foo: create or replace transient table db.from_shared_dbt_package.foo
    as (select 'ci' as c
    )
/* {"app": "dbt", "dbt_version": "1.11.5", "profile_name": "sf", "target_name": "ci", "node_id": "model.analytics.foo"} */;
07:03:28  1 of 1 OK created sql table model gsn_from_pkg.foo ............................. [SUCCESS 1 in 1.25s]
```

As we can see, the model is now being built into the schema `from_shared_dbt_package` instead - therefore the gsn logic in the dbt package `shared_pkg` is now being applied to our root dbt project.
