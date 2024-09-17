---
---

## Changing the materialization type based on the name of the model

```sql
-- models/vw_foo.sql
{{
    config(
        materialized=('view' if this.name.startswith('vw') else 'table')
    )
}}

select 1 id

-- models/tbl_bar.sql
{{
    config(
        materialized=('view' if this.name.startswith('vw') else 'table')
    )
}}

select 1 id
```

```sh
$ dbt run

20:04:58  Running with dbt=1.8.5
20:04:58  Registered adapter: postgres=1.8.2
20:04:58  Found 2 models, 417 macros
20:04:58  
20:04:58  Concurrency: 4 threads (target='pg')
20:04:58  
20:04:58  1 of 2 START sql table model public.tbl_bar .................................... [RUN]
20:04:58  2 of 2 START sql view model public.vw_foo ...................................... [RUN]
20:04:58  1 of 2 OK created sql table model public.tbl_bar ............................... [SELECT 1 in 0.07s]
20:04:58  2 of 2 OK created sql view model public.vw_foo ................................. [CREATE VIEW in 0.07s]
20:04:58  
20:04:58  Finished running 1 table model, 1 view model in 0 hours 0 minutes and 0.20 seconds (0.20s).
20:04:58  
20:04:58  Completed successfully
20:04:58  
20:04:58  Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```

It's not possible to have this logic be DRY (Don't Repeat Yourself) (e.g. specify this in the `dbt_project.yml` or macro-ify this logic) - we need to have that logic be in each and every model's config block. If we don't want to copy the same logic multiple times, we'll have to create a custom materialization for this.
