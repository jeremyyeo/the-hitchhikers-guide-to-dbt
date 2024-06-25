---
---

## dbt unit testing

https://docs.getdbt.com/docs/build/unit-tests

All examples below use a `dbt_project.yml` file like so unless specified:

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
config-version: 2
version: "1.0.0"

models:
 my_dbt_project:
    +materialized: table
```

### Expect output to have no rows

https://docs.getdbt.com/reference/resource-properties/unit-test-input

```sql
-- models/foo.sql
select 1 id
union 
select 2 id

-- models/bar.sql
select * from {{ ref('foo') }}
 where id != 1
```

```yaml
unit_tests:
  - name: test_is_valid
    model: bar
    given:
      - input: ref('foo')
        rows:
          - {id: 1}
    expect:
      rows: []
```

```sh
$ dbt test

04:09:39  Running with dbt=1.8.2
04:09:40  Registered adapter: postgres=1.8.1
04:09:40  Found 2 models, 413 macros, 1 unit test
04:09:40  
04:09:40  Concurrency: 4 threads (target='pg')
04:09:40  
04:09:40  1 of 1 START unit_test bar::test_is_valid ...................................... [RUN]
04:09:40  1 of 1 PASS bar::test_is_valid ................................................. [PASS in 0.10s]
04:09:40  
04:09:40  Finished running 1 unit test in 0 hours 0 minutes and 0.20 seconds (0.20s).
04:09:40  
04:09:40  Completed successfully
04:09:40  
04:09:40  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
```