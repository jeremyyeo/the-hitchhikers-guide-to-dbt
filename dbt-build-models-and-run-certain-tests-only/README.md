---
---

# dbt build models and run certain tests only

Sometimes, we want to:

1. Use the `dbt build` command.

> Note: By using `dbt build`, **all** tests of any selected models will run, even if your selection includes only specifically tagged tests.

2. Want only some tests to run but not all tests - i.e. only "critical" tests.

3. We don't want to go tagging each and every "noncritical" test - e.g:

```yaml
# models:
models:
  - name: foo
    columns:
      - name: c
        tests:
          - not_null:
              config:
                tags: ["critical"]
          - unique:
              config:
                tags: ["noncritical"]
  - name: bar
    columns:
      - name: c
        tests:
          - not_null:
              config:
                tags: ["noncritical"]
```

> By tagging tests, we can exclude them via `--exclude tag:noncritical` but if we have very many non critical tests, then we have to add these tags many times. Tags are also [additive (instead of clobber)](https://docs.getdbt.com/reference/define-configs#combining-configs) - so there is no "simple" way to set these via `dbt_project.yml`.

4. We don't want to modify test `severity`.

---

This is the simplest way to achieve all of the above.

```yaml
# dbt_project.yml
name: analytics
profile: all
version: "1.0.0"

models:
  analytics:
    +materialized: table

# All tests are disabled by default.
# Critical tests have config.enabled overwritten to `true` where they are declared - their schema.yml files.
tests:
  analytics:
    +enabled: "{{ true if var('all_tests', 0) == 1 else false | as_bool }}"

# models/schema.yml
models:
  - name: foo
    columns:
      - name: c
        tests:
          - not_null:
              config:
                enabled: true
          - unique
  - name: bar
    columns:
      - name: c
        tests:
          - not_null
```

```sql
-- models/foo.sql
select 1 c

-- models/bar.sql
select * from {{ ref('foo') }}
```

When we just want to build and have critical tests run:

```sh
$ dbt build

23:51:34  Running with dbt=1.10.5
23:51:34  Registered adapter: postgres=1.9.0
23:51:34  Unable to do partial parsing because config vars, config profile, or config target have changed
23:51:35  Found 2 models, 1 test, 549 macros
23:51:35
23:51:35  Concurrency: 1 threads (target='pg')
23:51:35
23:51:35  1 of 3 START sql table model public.foo ........................................ [RUN]
23:51:35  1 of 3 OK created sql table model public.foo ................................... [SELECT 1 in 0.05s]
23:51:35  2 of 3 START test not_null_foo_c ............................................... [RUN]
23:51:35  2 of 3 PASS not_null_foo_c ..................................................... [PASS in 0.02s]
23:51:35  3 of 3 START sql table model public.bar ........................................ [RUN]
23:51:35  3 of 3 OK created sql table model public.bar ................................... [SELECT 1 in 0.05s]
23:51:35
23:51:35  Finished running 2 table models, 1 test in 0 hours 0 minutes and 0.25 seconds (0.25s).
23:51:35
23:51:35  Completed successfully
23:51:35
23:51:35  Done. PASS=3 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=3
```

And of course, when we want all tests to run:

```sh
$ dbt build --vars '{"all_tests": 1}'

23:51:59  Running with dbt=1.10.5
23:51:59  Registered adapter: postgres=1.9.0
23:51:59  Unable to do partial parsing because config vars, config profile, or config target have changed
23:52:00  Found 2 models, 3 data tests, 549 macros
23:52:00
23:52:00  Concurrency: 1 threads (target='pg')
23:52:00
23:52:00  1 of 5 START sql table model public.foo ........................................ [RUN]
23:52:00  1 of 5 OK created sql table model public.foo ................................... [SELECT 1 in 0.05s]
23:52:00  2 of 5 START test not_null_foo_c ............................................... [RUN]
23:52:00  2 of 5 PASS not_null_foo_c ..................................................... [PASS in 0.02s]
23:52:00  3 of 5 START test unique_foo_c ................................................. [RUN]
23:52:00  3 of 5 PASS unique_foo_c ....................................................... [PASS in 0.01s]
23:52:00  4 of 5 START sql table model public.bar ........................................ [RUN]
23:52:00  4 of 5 OK created sql table model public.bar ................................... [SELECT 1 in 0.05s]
23:52:00  5 of 5 START test not_null_bar_c ............................................... [RUN]
23:52:00  5 of 5 PASS not_null_bar_c ..................................................... [PASS in 0.01s]
23:52:00
23:52:00  Finished running 2 table models, 3 data tests in 0 hours 0 minutes and 0.28 seconds (0.28s).
23:52:00
23:52:00  Completed successfully
23:52:00
23:52:00  Done. PASS=5 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=5
```

This means that you only have to manually override `config.enabled` on what few "critical" tests you have in your project (which I'm assuming the project has less off).

P.s. I try not to use bool/truthy/falsy types in my cli var passing due to https://gist.github.com/jeremyyeo/e97dbc79b536e2ae4a72d734fedb1812 which is why I'm using `1/0` here. However, if you're happy to use bool, then it can be simplified to `{{ var('all_tests', false) | as_bool }}` which we then pass via cli with `--vars '{"all_tests": true}'`.
