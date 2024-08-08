---
---

## Snowflake python models shared functions

An example of sharing Python functions across different dbt Python models.

### Making the shared Python code available in Snowflake

Here, the function we want to share across our different dbt models are in a python file `utils.py`.

```python
# utils.py
def double(x: int) -> int:
    """Doubles a number."""
    return x * 2
```

In order to make that available for use in Snowflake, we will be adding that file to a GitHub repository (https://github.com/jeremyyeo/snowpark-shared) and using Snowflake's "Git repository" functionality to connect to our GitHub repo (https://docs.snowflake.com/en/sql-reference/sql/create-git-repository).

> There's other ways to get these Python files onto Snowflake - such as uploading files into a Snowflake stage - but that is not the main focus of this guide.

```sql
-- create the api integration object.
create or replace api integration jyeo_git_api_integration
api_provider = git_https_api
api_allowed_prefixes = ('https://github.com/jeremyyeo')
enabled = true;

-- create the git repository object.
use schema development_jyeo.repositories;
create or replace git repository jyeo_repo_snowpark_shared
api_integration = jyeo_git_api_integration
origin = 'https://github.com/jeremyyeo/snowpark-shared.git';

-- fetch from the repo.
alter git repository jyeo_repo_snowpark_shared fetch;

-- inspect the files in the main branch of the repo.
ls @development_jyeo.repositories.jyeo_repo_snowpark_shared/branches/main;
```

![alt text](image.png)

^ As we can see `utils.py` exist in our main branch - which we can now import into our Python model.

### Importing our function from the shared Python file

Now that the Python file `utils.py` is available in Snowflake - we can go ahead and import and use it in our dbt Python model.

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
select 1 v

-- models/bar.sql
select 2 v
```

```python
# models/foo_double.py
from utils import double

def model(dbt, session):
    dbt.config(
        imports = ["@development_jyeo.repositories.jyeo_repo_snowpark_shared/branches/main/utils.py"]
    )
    old_df = dbt.ref("foo").to_pandas()
    old_df["V_MODIFIED"] = old_df["V"].apply(double)
    new_df = old_df[["V_MODIFIED"]]
    return new_df

# models/bar_double.py
from utils import double

def model(dbt, session):
    dbt.config(
        imports = ["@development_jyeo.repositories.jyeo_repo_snowpark_shared/branches/main/utils.py"]
    )
    old_df = dbt.ref("bar").to_pandas()
    old_df["V_MODIFIED"] = old_df["V"].apply(double)
    new_df = old_df[["V_MODIFIED"]]
    return new_df
```

In the example above - we're importing the `double` function from `utils.py` which we're then using in our Python model business logic.

```sh
$ dbt run

02:39:23  Running with dbt=1.8.4
02:39:24  Registered adapter: snowflake=1.8.3
02:39:24  Found 4 models, 560 macros
02:39:24  
02:39:29  Concurrency: 1 threads (target='sf')
02:39:29  
02:39:29  1 of 4 START sql table model dbt_jyeo.bar ...................................... [RUN]
02:39:32  1 of 4 OK created sql table model dbt_jyeo.bar ................................. [SUCCESS 1 in 3.05s]
02:39:32  2 of 4 START sql table model dbt_jyeo.foo ...................................... [RUN]
02:39:35  2 of 4 OK created sql table model dbt_jyeo.foo ................................. [SUCCESS 1 in 2.88s]
02:39:35  3 of 4 START python table model dbt_jyeo.bar_double ............................ [RUN]
02:39:45  3 of 4 OK created python table model dbt_jyeo.bar_double ....................... [SUCCESS 1 in 10.09s]
02:39:45  4 of 4 START python table model dbt_jyeo.foo_double ............................ [RUN]
02:39:54  4 of 4 OK created python table model dbt_jyeo.foo_double ....................... [SUCCESS 1 in 9.32s]
02:39:54  
02:39:54  Finished running 4 table models in 0 hours 0 minutes and 30.35 seconds (30.35s).
02:39:54  
02:39:54  Completed successfully
02:39:54  
02:39:54  Done. PASS=4 WARN=0 ERROR=0 SKIP=0 TOTAL=4
```

And to confirm things worked:

```sql
select * from development_jyeo.dbt_jyeo.foo_double
union
select * from development_jyeo.dbt_jyeo.bar_double
```

![alt text](image-1.png)

Both values have been doubled successfully.