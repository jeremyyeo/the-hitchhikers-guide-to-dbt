---
---

## dbt variables and environment variables

* https://docs.getdbt.com/docs/build/project-variables
* https://docs.getdbt.com/docs/build/environment-variables

### Variables

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
config-version: 2
version: "1.0.0"

models:
   my_dbt_project:
      +materialized: table

vars:
    my_var: some_value
```

```sql
-- models/foo.sql
select '{{ var("my_var") }}' as c
```

```sh
$ dbt compile -s foo
03:07:56  Running with dbt=1.8.4
03:07:56  Registered adapter: postgres=1.8.2
03:07:56  Unable to do partial parsing because config vars, config profile, or config target have changed
03:07:56  Found 1 model, 529 macros
03:07:56  
03:07:56  Concurrency: 4 threads (target='pg')
03:07:56  
03:07:56  Compiled node 'foo' is:
-- models/foo.sql
select 'some_value' as c
```

^ Here the variable `my_var` was set to a string `"some_value"` in the `dbt_project.yml` file. When we compile our model, `{{ var("my_var") }}` was swapped out with it's value `some_value`.

With the same setup above, we can override the var during run time:

```sh
$ dbt compile -s foo --vars 'my_var: not_some_value'
03:11:23  Running with dbt=1.8.4
03:11:23  Registered adapter: postgres=1.8.2
03:11:23  Unable to do partial parsing because config vars, config profile, or config target have changed
03:11:24  Found 1 model, 529 macros
03:11:24  
03:11:24  Concurrency: 4 threads (target='pg')
03:11:24  
03:11:24  Compiled node 'foo' is:
-- models/foo.sql
select 'not_some_value' as c
```

What happens when a var isn't declared in either in the `dbt_project.yml` nor during run time:

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

```sql
-- models/foo.sql
select '{{ var("my_var") }}' as c
```

```sh
$ dbt compile -s foo
03:14:37  Running with dbt=1.8.4
03:14:37  Registered adapter: postgres=1.8.2
03:14:37  Found 1 model, 529 macros
03:14:37  
03:14:37  Concurrency: 4 threads (target='pg')
03:14:37  
03:14:37  Encountered an error:
Runtime Error
  Compilation Error in model foo (models/foo.sql)
    Required var 'my_var' not found in config:
    Vars supplied to foo = {}
```

dbt complains that you declared a var and it isn't found - this behaviour is also seen in prior versions (for example dbt 1.6):

```sh
$ dbt compile -s foo
03:15:05  Running with dbt=1.6.14
03:15:05  Registered adapter: postgres=1.6.14
03:15:05  Unable to do partial parsing because of a version mismatch
03:15:05  Found 1 model, 0 sources, 0 exposures, 0 metrics, 468 macros, 0 groups, 0 semantic models
03:15:05  
03:15:06  Concurrency: 4 threads (target='pg')
03:15:06  
03:15:06  Encountered an error:
Runtime Error
  Compilation Error in model foo (models/foo.sql)
    Required var 'my_var' not found in config:
    Vars supplied to foo = {}
```

When attempting to use variables that can be not set in either the dbt_project.yml or during run time arg, we can set a default value:

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

```sql
-- models/foo.sql
select '{{ var("my_var", "take_this_instead") }}' as c
```

```sh
$ dbt compile -s foo
03:17:16  Running with dbt=1.8.4
03:17:16  Registered adapter: postgres=1.8.2
03:17:17  Found 1 model, 529 macros
03:17:17  
03:17:17  Concurrency: 4 threads (target='pg')
03:17:17  
03:17:17  Compiled node 'foo' is:
-- models/foo.sql
select 'take_this_instead' as c

$ dbt compile -s foo --vars 'my_var: dont_take_that'
03:18:10  Running with dbt=1.8.4
03:18:10  Registered adapter: postgres=1.8.2
03:18:10  Unable to do partial parsing because config vars, config profile, or config target have changed
03:18:10  Found 1 model, 529 macros
03:18:10  
03:18:11  Concurrency: 4 threads (target='pg')
03:18:11  
03:18:11  Compiled node 'foo' is:
-- models/foo.sql
select 'dont_take_that' as c
```

As we can see here - when we do:

```
              VVV Default value.
var("my_var", "take_this_instead")
    ^^^ Variable name.
```

We would resolve the var to be `"take_this_instead"` whenever dbt detects that the variable is unset.

### Environment variables

Environment variables have the exact same behaviour as vars above. They will error if unset and there is no default value:

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

```sql
-- models/foo.sql
select '{{ env_var("my_env_var") }}' as c
```

```sh
$ unset my_env_var
$ echo $my_env_var # Nothing to echo cause is unset.

$ dbt compile -s foo
03:23:06  Running with dbt=1.8.4
03:23:06  Registered adapter: postgres=1.8.2
03:23:06  Unable to do partial parsing because config vars, config profile, or config target have changed
03:23:07  Encountered an error:
Parsing Error
  Env var required but not provided: 'my_env_var'
```

```sh
$ export my_env_var=some_value
$ echo $my_env_var # Now we have a value so we have something to echo.
some_value
$ dbt compile -s foo
03:25:21  Running with dbt=1.8.4
03:25:21  Registered adapter: postgres=1.8.2
03:25:21  Unable to do partial parsing because config vars, config profile, or config target have changed
03:25:22  Found 1 model, 529 macros
03:25:22  
03:25:22  Concurrency: 4 threads (target='pg')
03:25:22  
03:25:22  Compiled node 'foo' is:
-- models/foo.sql
select 'some_value' as c
```

And again, as vars do, we can set a default so as to not error if an env var is unset:

```sql
-- models/foo.sql
select '{{ env_var("my_env_var", "take_this_instead") }}' as c
```

```sh
$ unset my_env_var
$ echo my_env_var
$ dbt compile -s foo
03:27:20  Running with dbt=1.8.4
03:27:20  Registered adapter: postgres=1.8.2
03:27:20  Found 1 model, 529 macros
03:27:20  
03:27:20  Concurrency: 4 threads (target='pg')
03:27:20  
03:27:20  Compiled node 'foo' is:
-- models/foo.sql
select 'take_this_instead' as c
```

### Env vars in vars

A common pattern is to set the default value of vars to env vars - in cases when you only occasionally want the var value to be overridden to something else:

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

```sql
-- models/foo.sql
select '{{ var("my_var", env_var("my_env_var")) }}' as c
```

```sh
$ export my_env_var=from_env_var
$ dbt compile -s foo
03:30:16  Running with dbt=1.8.4
03:30:16  Registered adapter: postgres=1.8.2
03:30:16  Found 1 model, 529 macros
03:30:16  
03:30:17  Concurrency: 4 threads (target='pg')
03:30:17  
03:30:17  Compiled node 'foo' is:
-- models/foo.sql
select 'from_env_var' as c

$ dbt compile -s foo --vars 'my_var: from_var_instead'
03:30:52  Running with dbt=1.8.4
03:30:52  Registered adapter: postgres=1.8.2
03:30:53  Unable to do partial parsing because config vars, config profile, or config target have changed
03:30:53  Found 1 model, 529 macros
03:30:53  
03:30:53  Concurrency: 4 threads (target='pg')
03:30:53  
03:30:53  Compiled node 'foo' is:
-- models/foo.sql
select 'from_var_instead' as c
```

### Setting configs using env vars

Here we want to set the `full_refresh` config to be `false` by default using a combination of env var and var.

```yaml
# dbt_project.yml
name: my_dbt_project
profile: all
config-version: 2
version: "1.0.0"

models:
   my_dbt_project:
      +materialized: table
      +full_refresh: '{{ (var("var_full_refresh_allowed", env_var("ENV_FULL_REFRESH_ALLOWED", "blocked")) == "allowed") | as_bool }}'
```

> Note:  My logic is attempting to match on the string `"allowed"` instead of something like `True` or `true` because there are a lot of gotchas when you use boolean types to do so: https://gist.github.com/jeremyyeo/e97dbc79b536e2ae4a72d734fedb1812 

```sql
-- models/foo.sql
{{ config(materialized='incremental') }}

select 1 id
```

First we make sure our incremental table doesn't exist yet and then we build our model for the first time.

> Logs below will be truncated or else they will be unnecessarily long.

```sh
$ unset ENV_FULL_REFRESH_ALLOWED
$ echo $ENV_FULL_REFRESH_ALLOWED
$ dbt --debug run

04:04:16  1 of 1 START sql incremental model public.foo .................................. [RUN]
04:04:16  Re-using an available connection from the pool (formerly list_postgres_public, now model.my_dbt_project.foo)
04:04:16  Began compiling node model.my_dbt_project.foo
04:04:16  Writing injected SQL for node "model.my_dbt_project.foo"
04:04:16  Began executing node model.my_dbt_project.foo
04:04:16  Writing runtime sql for node "model.my_dbt_project.foo"
04:04:16  Using postgres connection "model.my_dbt_project.foo"
04:04:16  On model.my_dbt_project.foo: BEGIN
04:04:16  Opening a new connection, currently in state closed
04:04:16  SQL status: BEGIN in 0.0 seconds
04:04:16  Using postgres connection "model.my_dbt_project.foo"
04:04:16  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
  create  table "postgres"."public"."foo"
    as
  (
    -- models/foo.sql
select 1 id
  );
04:04:16  SQL status: SELECT 1 in 0.0 seconds
04:04:16  On model.my_dbt_project.foo: COMMIT
04:04:16  Using postgres connection "model.my_dbt_project.foo"
04:04:16  On model.my_dbt_project.foo: COMMIT
04:04:16  SQL status: COMMIT in 0.0 seconds
04:04:16  On model.my_dbt_project.foo: Close
04:04:16  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'b5249ec0-c2f9-46f3-8048-e7ab76a8f2e5', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1055a3bd0>]}
04:04:16  1 of 1 OK created sql incremental model public.foo ............................. [SELECT 1 in 0.05s]

$ dbt --debug run
04:04:48  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
      insert into "postgres"."public"."foo" ("id")
    (
        select "id"
        from "foo__dbt_tmp160448270582"
    )
04:04:48  SQL status: INSERT 0 1 in 0.0 seconds
04:04:48  On model.my_dbt_project.foo: COMMIT
04:04:48  Using postgres connection "model.my_dbt_project.foo"
04:04:48  On model.my_dbt_project.foo: COMMIT
04:04:48  SQL status: COMMIT in 0.0 seconds
04:04:48  On model.my_dbt_project.foo: Close
04:04:48  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '4f26aa54-7952-439f-b6f8-503ee03c23e2', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x114739750>]}
04:04:48  1 of 1 OK created sql incremental model public.foo ............................. [INSERT 0 1 in 0.09s]
```

Here, I ran it twice and we can see that the subsequent (incremental) run did an insert, as expected. What happens if we attempt to do a full refresh.

```sh
$ dbt --debug run --full-refresh

04:08:02  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */

      insert into "postgres"."public"."foo" ("id")
    (
        select "id"
        from "foo__dbt_tmp160802365778"
    )

  
04:08:02  SQL status: INSERT 0 1 in 0.0 seconds
04:08:02  On model.my_dbt_project.foo: COMMIT
04:08:02  Using postgres connection "model.my_dbt_project.foo"
04:08:02  On model.my_dbt_project.foo: COMMIT
04:08:02  SQL status: COMMIT in 0.0 seconds
04:08:02  On model.my_dbt_project.foo: Close
04:08:02  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '5ce26489-9c82-4c31-ad45-38aa8af5827b', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x107c4bbd0>]}
04:08:02  1 of 1 OK created sql incremental model public.foo ............................. [INSERT 0 1 in 0.08s]
```

Normally, when we add a `--full-refresh` flag, the model will be created from scratch (i.e. `create table ...`). However, if we look at our `+full_refresh` config:

```
'{{ (var("var_full_refresh_allowed", env_var("ENV_FULL_REFRESH_ALLOWED", "blocked")) == "allowed") | as_bool }}'
```

1. The var `var_full_refresh_allowed` is not set - therefore we fall into resolving its default value.
2. The default value is itself an env var `ENV_FULL_REFRESH_ALLOWED`.
3. The env var `ENV_FULL_REFRESH_ALLOWED` is also unset - therefore it resolves to its default value `"blocked"`.
4. Thus `"blocked" == "allowed" | as_bool` resolves to `false`.

Since the `full_refresh` config of the model is `false` - then our model isn't allowed to be fully refreshed - even WITH the `--full-refresh` flag added to our run command.

> Note that the outcome above would be the same as if we had set the env var `ENV_FULL_REFRESH_ALLOWED` to be any string besides `"allowed"`... for example `export ENV_FULL_REFRESH_ALLOWED=blocked` or `export ENV_FULL_REFRESH_ALLOWED=nope`.

How can we then permit the model to be fully refreshed? We can set the var `var_full_refresh_allowed` during runtime:

```sh
$ dbt --debug run --full-refresh --vars 'var_full_refresh_allowed: allowed'
04:14:29  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
  create  table "postgres"."public"."foo__dbt_tmp"
    as
  (
    -- models/foo.sql
select 1 id
  );
  
04:14:29  SQL status: SELECT 1 in 0.0 seconds
04:14:29  Using postgres connection "model.my_dbt_project.foo"
04:14:29  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
alter table "postgres"."public"."foo" rename to "foo__dbt_backup"
04:14:29  SQL status: ALTER TABLE in 0.0 seconds
04:14:29  Using postgres connection "model.my_dbt_project.foo"
04:14:29  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
alter table "postgres"."public"."foo__dbt_tmp" rename to "foo"
04:14:29  SQL status: ALTER TABLE in 0.0 seconds
04:14:29  On model.my_dbt_project.foo: COMMIT
04:14:29  Using postgres connection "model.my_dbt_project.foo"
04:14:29  On model.my_dbt_project.foo: COMMIT
04:14:29  SQL status: COMMIT in 0.0 seconds
04:14:29  Applying DROP to: "postgres"."public"."foo__dbt_backup"
04:14:29  Using postgres connection "model.my_dbt_project.foo"
04:14:29  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
drop table if exists "postgres"."public"."foo__dbt_backup" cascade
04:14:29  SQL status: DROP TABLE in 0.0 seconds
04:14:29  On model.my_dbt_project.foo: Close
04:14:29  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '9a47d265-cc2f-4521-93cf-c3d612003ff8', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x1059faf50>]}
04:14:29  1 of 1 OK created sql incremental model public.foo ............................. [SELECT 1 in 0.06s]
```

Or we can set the env var `ENV_FULL_REFRESH_ALLOWED`:

```sh
$ export ENV_FULL_REFRESH_ALLOWED=allowed
$ dbt --debug run
04:16:34  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
  create  table "postgres"."public"."foo__dbt_tmp"
    as
  (
    -- models/foo.sql
select 1 id
  );
04:16:34  SQL status: SELECT 1 in 0.0 seconds
04:16:34  Using postgres connection "model.my_dbt_project.foo"
04:16:34  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
alter table "postgres"."public"."foo" rename to "foo__dbt_backup"
04:16:34  SQL status: ALTER TABLE in 0.0 seconds
04:16:34  Using postgres connection "model.my_dbt_project.foo"
04:16:34  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
alter table "postgres"."public"."foo__dbt_tmp" rename to "foo"
04:16:34  SQL status: ALTER TABLE in 0.0 seconds
04:16:34  On model.my_dbt_project.foo: COMMIT
04:16:34  Using postgres connection "model.my_dbt_project.foo"
04:16:34  On model.my_dbt_project.foo: COMMIT
04:16:34  SQL status: COMMIT in 0.0 seconds
04:16:34  Applying DROP to: "postgres"."public"."foo__dbt_backup"
04:16:34  Using postgres connection "model.my_dbt_project.foo"
04:16:34  On model.my_dbt_project.foo: /* {"app": "dbt", "dbt_version": "1.8.4", "profile_name": "all", "target_name": "pg", "node_id": "model.my_dbt_project.foo"} */
drop table if exists "postgres"."public"."foo__dbt_backup" cascade
04:16:34  SQL status: DROP TABLE in 0.0 seconds
04:16:34  On model.my_dbt_project.foo: Close
04:16:34  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': '2eda4a27-b15d-4a5e-bc11-743c5bb1bda8', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x107ba9cd0>]}
04:16:34  1 of 1 OK created sql incremental model public.foo ............................. [SELECT 1 in 0.06s]
```