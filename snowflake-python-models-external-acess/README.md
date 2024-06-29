---
---

## Snowflake python models external access

Video: https://youtu.be/mZ2IhFXeAUw 

Docs:
* https://docs.getdbt.com/docs/build/python-models#specific-data-platforms
* https://docs.snowflake.com/en/sql-reference/sql/create-external-access-integration

An example Snowflake Python model that access an external API / endpoint to retrieve some data.

For this exercise, I've created a test endpoint on [Pipedream](https://pipedream.com/) that will return some message if the Authorization Bearer token is valid:

```python
def handler(pd: "pipedream"):
  auth_check = pd.steps["trigger"]["event"]["headers"].get("authorization")
  if auth_check == "Bearer mellon":
    pd.respond({"status": 200, "body": {"message": "Speak friend and enter."}})
  else:
    pd.respond({"status": 403, "body": {"message": "You shall not pass."}})
```

```sh
$ curl https://eoqpffa3vohq4qp.m.pipedream.net
{"message": "You shall not pass."}

$ curl -H "Authorization: Bearer mellon" https://eoqpffa3vohq4qp.m.pipedream.net 
{"message": "Speak friend and enter."}
```

### Testing access to this API directly from Snowflake

The first thing we should test, is to access this endpoint from Snowflake directly and see how things work. There's a couple of things we need to do:

1. Create a secret.

```sql
use role accountadmin;

create or replace secret development_jyeo.dbt_jyeo.my_api_token 
  type = generic_string 
  secret_string = 'mellon'
;

grant read on secret development_jyeo.dbt_jyeo.my_api_token to role transformer;
```

2. Create a network rule and an external access integration.

```sql
create or replace network rule pipedream_network_rule
  mode = egress
  type = host_port
  value_list = ('eoqpffa3vohq4qp.m.pipedream.net')
;

create or replace external access integration pipedream_access_integration
  allowed_network_rules = (pipedream_network_rule)
  allowed_authentication_secrets = (my_api_token)
  enabled = true
;

grant usage on integration pipedream_access_integration to role transformer;
```

3. Create a function that uses all the above components:

```sql
use role transformer;

create or replace function my_api_function()
returns string
language python
runtime_version = 3.8
handler = 'main'
external_access_integrations = (pipedream_access_integration)
packages = ('snowflake-snowpark-python','requests')
secrets = ('cred' = my_api_token )
as
$$
import _snowflake
import requests
import json
session = requests.Session()
def main():
  token = _snowflake.get_generic_secret_string('cred')
  url = "https://eoqpffa3vohq4qp.m.pipedream.net"
  response = session.get(url, headers = {"Authorization": "Bearer " + token})
  return response.json()
$$;
```
> Note: Notice how there's a lot of things and configs that we need here (e.g. defining `packages`, `external_access_integrations` and what not) - running Python in Snowflake is not the same as running a simple Python script on your own machine.

Finally, call our function and look at the results:

```sql
select my_api_function() as gandalf_says;
```

```
+----------------------------------------+
| gandalf_says                           |
+========================================+
| {"message": "Speak friend and enter."} |
+----------------------------------------+
```

### Testing access to this API from a dbt Python model

Now that we know that we're able to access the API, let's use what we've learned above in a dbt Python model:

> Note: A Python model is **not the same** as simply running a Python script on your own machine. So if you wrote a simply Python script that is able to access the API just fine - **do not blindly assume** that things will work without issues when you simply copy paste the code into your Python model. 

```python
# models/gandalf_says.py
import pandas
import snowflake.snowpark as snowpark

def model(dbt, session: snowpark.Session):
    dbt.config(
        materialized = "table",
        packages = ["snowflake-snowpark-python", "requests"],
        secrets={"cred": "my_api_token"},
        external_access_integrations=["pipedream_access_integration"],
        python_version = "3.8"
    )
    import _snowflake
    import requests
    import json
    r = requests.Session()
    token = _snowflake.get_generic_secret_string('cred')
    url = "https://eoqpffa3vohq4qp.m.pipedream.net"
    response = r.get(url, headers = {"Authorization": "Bearer " + token})
    return session.create_dataframe(
        pandas.DataFrame(
            [response.json()]
        )
    )
```

```sh
$ dbt --debug build -s gandalf_says
00:46:01  Sending event: {'category': 'dbt', 'action': 'invocation', 'label': 'start', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b829f90>, <snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b86ab90>, <snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b88ded0>]}
00:46:01  Running with dbt=1.8.2
00:46:01  running dbt with arguments {'printer_width': '80', 'indirect_selection': 'eager', 'log_cache_events': 'False', 'write_json': 'True', 'partial_parse': 'True', 'cache_selected_only': 'False', 'profiles_dir': '/Users/jeremy/.dbt', 'fail_fast': 'False', 'version_check': 'True', 'log_path': '/Users/jeremy/git/dbt-basic/logs', 'debug': 'True', 'warn_error': 'None', 'use_colors': 'True', 'use_experimental_parser': 'False', 'no_print': 'None', 'quiet': 'False', 'log_format': 'default', 'introspect': 'True', 'static_parser': 'True', 'invocation_command': 'dbt --debug build -s gandalf_says', 'target_path': 'None', 'warn_error_options': 'WarnErrorOptions(include=[], exclude=[])', 'send_anonymous_usage_stats': 'True'}
00:46:02  Sending event: {'category': 'dbt', 'action': 'project_id', 'label': 'a0e817a0-5094-450f-9d54-71f1faee763f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b8a1350>]}
00:46:02  Sending event: {'category': 'dbt', 'action': 'adapter_info', 'label': 'a0e817a0-5094-450f-9d54-71f1faee763f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b1ae650>]}
00:46:02  Registered adapter: snowflake=1.8.3
00:46:02  checksum: f0a141b9de567cb7f131613ba2682f40b3a1c6a157f81c6e5435862eff68f109, vars: {}, profile: , target: , version: 1.8.2
00:46:02  Unable to do partial parsing because a project config has changed
00:46:02  Sending event: {'category': 'dbt', 'action': 'partial_parser', 'label': 'a0e817a0-5094-450f-9d54-71f1faee763f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10903c050>]}
00:46:02  Sending event: {'category': 'dbt', 'action': 'load_project', 'label': 'a0e817a0-5094-450f-9d54-71f1faee763f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10901c610>]}
00:46:02  Sending event: {'category': 'dbt', 'action': 'resource_counts', 'label': 'a0e817a0-5094-450f-9d54-71f1faee763f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x15c3a0250>]}
00:46:02  Found 1 model, 446 macros
00:46:02  
00:46:02  Acquiring new snowflake connection 'master'
00:46:02  Acquiring new snowflake connection 'list_development_jyeo'
00:46:02  Using snowflake connection "list_development_jyeo"
00:46:02  On list_development_jyeo: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "connection_name": "list_development_jyeo"} */
show terse schemas in database development_jyeo
    limit 10000
00:46:02  Opening a new connection, currently in state init
00:46:04  SQL status: SUCCESS 44 in 1.0 seconds
00:46:04  On list_development_jyeo: Close
00:46:04  Acquiring new snowflake connection 'list_development_jyeo_dbt_jyeo'
00:46:04  Using snowflake connection "list_development_jyeo_dbt_jyeo"
00:46:04  On list_development_jyeo_dbt_jyeo: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "connection_name": "list_development_jyeo_dbt_jyeo"} */
show objects in development_jyeo.dbt_jyeo limit 10000
00:46:04  Opening a new connection, currently in state init
00:46:05  SQL status: SUCCESS 250 in 1.0 seconds
00:46:05  On list_development_jyeo_dbt_jyeo: Close
00:46:06  Sending event: {'category': 'dbt', 'action': 'runnable_timing', 'label': 'a0e817a0-5094-450f-9d54-71f1faee763f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x15c04fc90>]}
00:46:06  Concurrency: 1 threads (target='sf')
00:46:06  
00:46:06  Began running node model.my_dbt_project.gandalf_says
00:46:06  1 of 1 START python table model dbt_jyeo.gandalf_says .......................... [RUN]
00:46:06  Re-using an available connection from the pool (formerly list_development_jyeo, now model.my_dbt_project.gandalf_says)
00:46:06  Began compiling node model.my_dbt_project.gandalf_says
00:46:06  Writing injected SQL for node "model.my_dbt_project.gandalf_says"
00:46:06  Began executing node model.my_dbt_project.gandalf_says
00:46:06  Writing runtime python for node "model.my_dbt_project.gandalf_says"
00:46:06  Using snowflake connection "model.my_dbt_project.gandalf_says"
00:46:06  On model.my_dbt_project.gandalf_says: /* {"app": "dbt", "dbt_version": "1.8.2", "profile_name": "all", "target_name": "sf", "node_id": "model.my_dbt_project.gandalf_says"} */
WITH gandalf_says__dbt_sp AS PROCEDURE ()

RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = '3.8'
PACKAGES = ('snowflake-snowpark-python', 'requests')
EXTERNAL_ACCESS_INTEGRATIONS = (pipedream_access_integration)
SECRETS = ('cred' = my_api_token)

HANDLER = 'main'
EXECUTE AS CALLER
AS
$$

import sys
sys._xoptions['snowflake_partner_attribution'].append("dbtLabs_dbtPython")


  
    
# gandalf_says.py
import pandas
import snowflake.snowpark as snowpark

def model(dbt, session: snowpark.Session):
    dbt.config(
        materialized = "table",
        packages = ["snowflake-snowpark-python", "requests"],
        secrets={"cred": "my_api_token"},
        external_access_integrations=["pipedream_access_integration"],
        python_version = "3.8"
    )
    import _snowflake
    import requests
    import json
    r = requests.Session()
    token = _snowflake.get_generic_secret_string('cred')
    url = "https://eoqpffa3vohq4qp.m.pipedream.net"
    response = r.get(url, headers = {"Authorization": "Bearer " + token})
    return session.create_dataframe(
        pandas.DataFrame(
            [response.json()]
        )
    )


# This part is user provided model code
# you will need to copy the next section to run the code
# COMMAND ----------
# this part is dbt logic for get ref work, do not modify

def ref(*args, **kwargs):
    refs = {}
    key = '.'.join(args)
    version = kwargs.get("v") or kwargs.get("version")
    if version:
        key += f".v{version}"
    dbt_load_df_function = kwargs.get("dbt_load_df_function")
    return dbt_load_df_function(refs[key])


def source(*args, dbt_load_df_function):
    sources = {}
    key = '.'.join(args)
    return dbt_load_df_function(sources[key])


config_dict = {}


class config:
    def __init__(self, *args, **kwargs):
        pass

    @staticmethod
    def get(key, default=None):
        return config_dict.get(key, default)

class this:
    """dbt.this() or dbt.this.identifier"""
    database = "development_jyeo"
    schema = "dbt_jyeo"
    identifier = "gandalf_says"
    
    def __repr__(self):
        return 'development_jyeo.dbt_jyeo.gandalf_says'


class dbtObj:
    def __init__(self, load_df_function) -> None:
        self.source = lambda *args: source(*args, dbt_load_df_function=load_df_function)
        self.ref = lambda *args, **kwargs: ref(*args, **kwargs, dbt_load_df_function=load_df_function)
        self.config = config
        self.this = this()
        self.is_incremental = False

# COMMAND ----------

# To run this in snowsight, you need to select entry point to be main
# And you may have to modify the return type to text to get the result back
# def main(session):
#     dbt = dbtObj(session.table)
#     df = model(dbt, session)
#     return df.collect()

# to run this in local notebook, you need to create a session following examples https://github.com/Snowflake-Labs/sfguide-getting-started-snowpark-python
# then you can do the following to run model
# dbt = dbtObj(session.table)
# df = model(dbt, session)


def materialize(session, df, target_relation):
    # make sure pandas exists
    import importlib.util
    package_name = 'pandas'
    if importlib.util.find_spec(package_name):
        import pandas
        if isinstance(df, pandas.core.frame.DataFrame):
          session.use_database(target_relation.database)
          session.use_schema(target_relation.schema)
          # session.write_pandas does not have overwrite function
          df = session.createDataFrame(df)
    
    df.write.mode("overwrite").save_as_table('development_jyeo.dbt_jyeo.gandalf_says', table_type='transient')

def main(session):
    dbt = dbtObj(session.table)
    df = model(dbt, session)
    materialize(session, df, dbt.this)
    return "OK"

  
$$
CALL gandalf_says__dbt_sp();
00:46:06  Opening a new connection, currently in state closed
00:46:12  SQL status: SUCCESS 1 in 6.0 seconds
00:46:12  On model.my_dbt_project.gandalf_says: Close
00:46:12  Sending event: {'category': 'dbt', 'action': 'run_model', 'label': 'a0e817a0-5094-450f-9d54-71f1faee763f', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x15cb08b90>]}
00:46:12  1 of 1 OK created python table model dbt_jyeo.gandalf_says ..................... [SUCCESS 1 in 6.53s]
00:46:12  Finished running node model.my_dbt_project.gandalf_says
00:46:12  Connection 'master' was properly closed.
00:46:12  Connection 'model.my_dbt_project.gandalf_says' was properly closed.
00:46:12  Connection 'list_development_jyeo_dbt_jyeo' was properly closed.
00:46:12  
00:46:12  Finished running 1 table model in 0 hours 0 minutes and 9.64 seconds (9.64s).
00:46:12  Command end result
00:46:12  
00:46:12  Completed successfully
00:46:12  
00:46:12  Done. PASS=1 WARN=0 ERROR=0 SKIP=0 TOTAL=1
00:46:12  Resource report: {"command_name": "build", "command_success": true, "command_wall_clock_time": 11.252702, "process_user_time": 2.992993, "process_kernel_time": 1.078264, "process_mem_max_rss": "202899456", "process_in_blocks": "0", "process_out_blocks": "0"}
00:46:12  Command `dbt build` succeeded at 12:46:12.639660 after 11.25 seconds
00:46:12  Sending event: {'category': 'dbt', 'action': 'invocation', 'label': 'end', 'context': [<snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b72bb50>, <snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x10b3bb390>, <snowplow_tracker.self_describing_json.SelfDescribingJson object at 0x105188450>]}
00:46:12  Flushing usage events
```

> Note: If you have an error - copy paste the whole code into Snowflake and start debugging from there. dbt does not do any computation here - it merely generates code and sends it to Snowflake for execution so you want to get it right in Snowflake first.

Then select from our table in Snowflake:

```sql
select * from development_jyeo.dbt_jyeo.gandalf_says;
```

```
+-------------------------+
| gandalf_says            |
+=========================+
| Speak friend and enter. |
+-------------------------+
```

Note that it's also possible to access the API using just the function we created previously in just a normal SQL model:

```sql
-- models/gandalf_says_alt.sql
select development_jyeo.dbt_jyeo.my_api_function() as gandalf_says
```

Since a model like this would simply result in dbt issuing a SQL command like:

```sql
create or replace transient table development_jyeo.dbt_jyeo.gandalf_says_alt
         as
        (-- models/gandalf_says_alt.sql
select development_jyeo.dbt_jyeo.my_api_function() as gandalf_says
        );
```

And we know that we can already call that function (`my_api_function()`) successfully as tested previously. So a dbt Python model is not strictly necessary to use external API's here.
