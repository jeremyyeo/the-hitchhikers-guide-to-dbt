---
---

## dbt warnings

https://docs.getdbt.com/reference/global-configs/warnings

A catalog of warnings and what they mean with examples.

### Notes

1. These warnings show up at the start of the logs even if your selection does not include the node that dbt is warning you about. For example, you may have done `dbt seed` or `dbt run --select foo` but dbt will still emit all the warnings even though the warnings were not relevant to seeds or model `foo`.
2. Due to partial parsing, warnings may show up in one run but not in a subsequent one even if the criteria for the warning has not yet been resolved - https://github.com/dbt-labs/dbt-core/issues/10323

### Configuration paths exist in your dbt_project.yml file which do not apply to any resources.

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
config-version: 2
version: "1.0.0"

models:
   my_dbt_project:
      +materialized: table
      marts:
         +materialized: view
```

```sql
-- models/foo.sql
select 1 id
```

```sh
$ dbt run
00:49:55  Running with dbt=1.6.14
00:49:55  Registered adapter: postgres=1.6.14
00:49:55  Unable to do partial parsing because saved manifest not found. Starting full parse.
00:49:55  [WARNING]: Configuration paths exist in your dbt_project.yml file which do not apply to any resources.
There are 1 unused configuration paths:
- models.my_dbt_project.marts
00:49:55  Found 1 model, 0 sources, 0 exposures, 0 metrics, 352 macros, 0 groups, 0 semantic models
00:49:55  
00:49:55  Concurrency: 4 threads (target='pg')
00:49:55  
00:49:55  1 of 1 START sql table model public.foo ........................................ [RUN]
00:49:55  1 of 1 OK created sql table model public.foo ................................... [SELECT 1 in 0.07s]
00:49:55  
00:49:55  Finished running 1 table model in 0 hours 0 minutes and 0.20 seconds (0.20s).
00:49:55  
00:49:55  Completed successfully
00:49:55  
00:49:55  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

^ Here I'm telling dbt to apply `materialized: view` to models in a folder `models/marts/` which does not exist and dbt is warning me of exactly that. The folder can simply not exist literally or because of incorrect casing - i.e. the actual folder name is `Marts` but you tried to apply the config on folder `marts`.

### Did not find matching node for patch with name '...' in the '...' section of file '....yml'

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
version: 2
models:
  - name: foo
    description: Lorem ipsum
  - name: bar
    description: Lorem ipsum
```

```sql
-- models/foo.sql
select 1 id
```

```sh
$ dbt run
01:21:31  Running with dbt=1.6.14
01:21:31  Registered adapter: postgres=1.6.14
01:21:31  Unable to do partial parsing because saved manifest not found. Starting full parse.
01:21:31  [WARNING]: Did not find matching node for patch with name 'bar' in the 'models' section of file 'models/schema.yml'
01:21:31  Found 1 model, 0 sources, 0 exposures, 0 metrics, 352 macros, 0 groups, 0 semantic models
01:21:31  
01:21:31  Concurrency: 4 threads (target='pg')
01:21:31  
01:21:31  1 of 1 START sql table model public.foo ........................................ [RUN]
01:21:31  1 of 1 OK created sql table model public.foo ................................... [SELECT 1 in 0.08s]
01:21:31  
01:21:31  Finished running 1 table model in 0 hours 0 minutes and 0.18 seconds (0.18s).
01:21:31  
01:21:31  Completed successfully
01:21:31  
01:21:31  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```

We told dbt that there exist a model `bar` in the `schema.yml` file but in reality - it does not exist. Note it can also not exist due to a mismatched case - i.e. in the `schema.yml` file we did `name: Bar` (uppercase "B") but the model is actually `bar.sql` (lowercase "b").
