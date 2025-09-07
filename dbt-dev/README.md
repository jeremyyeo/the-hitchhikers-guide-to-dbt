---
---

# dbt dev

Some self notes when working on dbt itself.

## dbt-adapters

https://github.com/dbt-labs/dbt-adapters

1. Adapter is for the implementation of "adapter" (i.e. data platform) specific functionality - e.g. the out of the box macros, vendor (e.g. Snowflake, BigQuery) python library methods.
2. Different data platforms support different functionality (e.g. Snowflake can clone tables but Postgres cannot).

### Setup

Official docs: https://github.com/dbt-labs/dbt-adapters/blob/main/CONTRIBUTING.md

```sh
# Clone and cd into repo.
git clone git@github.com:dbt-labs/dbt-adapters.git
cd dbt-adapters

# Install hatch in a new virtual environment.
# You probably don't need this if you have hatch
# installed globally but I don't as I like to keep
# my global environment free of python packages.
python -m venv venv
source venv/bin/activate
pip install --upgrade pip hatch

# Working on dbt-postgres example.
cd dbt-postgres
hatch config set dirs.env.virtual .hatch
hatch run setup

# At this stage, you'll want to modify the `test.env` file and put
# valid credentials (username, password, etc).

# Activate the virtual env that hatch created.
hatch shell

# Test if integration tests are working to validate connection to the Postgres database.
hatch run integration-tests
```

```sh
========================================================================= test session starts ==========================================================================
platform darwin -- Python 3.11.9, pytest-7.4.4, pluggy-1.6.0 -- /Users/jeremy/git/dbt-adapters/dbt-postgres/.hatch/dbt-postgres/bin/python
cachedir: .pytest_cache
rootdir: /Users/jeremy/git/dbt-adapters/dbt-postgres
configfile: pyproject.toml
plugins: ddtrace-2.3.0, xdist-3.8.0, mock-3.14.1, dotenv-0.5.2
11 workers [575 items]
scheduling tests via LoadScheduling

tests/functional/test_clean.py::TestCleanPathOutsideProjectWithFlag::test_clean_path_outside_project
tests/functional/test_access.py::TestAccess::test_access_attribute
tests/functional/test_config.py::TestProfileFile::test_env_vars_env_target
[gw3] [  0%] SKIPPED tests/functional/test_config.py::TestProfileFile::test_env_vars_env_target
...
ERROR tests/functional/basic/test_mixed_case_db.py::test_basic - dbt.adapters.exceptions.connection.FailedToConnectError: Database Error
================================================== 16 failed, 545 passed, 12 skipped, 24 warnings, 2 errors in 47.42s ==================================================
```

```sh
# Check to see if packages are install in "editable" mode
pip list | grep dbt

dbt-adapters              1.16.5      /Users/jeremy/git/dbt-adapters/dbt-adapters
dbt-common                1.28.0
dbt-core                  1.11.0a1
dbt-extractor             0.6.0
dbt-postgres              1.9.1a0     /Users/jeremy/git/dbt-adapters/dbt-postgres
dbt-protos                1.0.348
dbt-semantic-interfaces   0.9.0
dbt-tests-adapter         1.18.0      /Users/jeremy/git/dbt-adapters/dbt-tests-adapter

# Here we can see the various dbt packages installed in editable model by evidence of
# the path to their respective directories.
```
