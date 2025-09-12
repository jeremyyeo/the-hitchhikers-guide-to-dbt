---
---

# dbt dev

Some self notes when working on dbt itself.

## dbt-core

https://github.com/dbt-labs/dbt-core

### Setup

Official docs: https://github.com/dbt-labs/dbt-core/blob/main/CONTRIBUTING.md

```sh
git clone https://github.com/dbt-labs/dbt-core.git
cd dbt-core

# Install into in a new virtual environment.
python -m venv venv
source venv/bin/activate
make dev

# Test to see if things work.
dbt --version
```

```sh
Core:
  - installed: 1.11.0-a1
  - latest:    1.10.11   - Ahead of latest version!

Plugins:
  - postgres: 1.9.1a0 - Update available!

  At least one plugin is out of date with dbt-core.
  You can find instructions for upgrading here:
  https://docs.getdbt.com/docs/installation
```

```sh
# Try running the test suite.
make test
```

```sh
py: install_deps> python -I -m pip install -r dev-requirements.txt -r editable-requirements.txt
py: commands[0]> .tox/py/bin/python -m pytest --cov=core --cov-report=xml tests/unit
===================================================== test session starts ======================================================
platform darwin -- Python 3.11.9, pytest-7.4.4, pluggy-1.6.0
cachedir: .tox/py/.pytest_cache
rootdir: /Users/jeremy/git/dbt-core
configfile: pytest.ini
plugins: csv-3.0.0, flaky-3.8.1, xdist-3.8.0, mock-3.15.0, hypothesis-6.138.15, dotenv-0.5.2, ddtrace-2.21.3, split-0.10.0, cov-7.0.0
collected 1522 items

tests/unit/test_artifact_upload.py ...............                                                                       [  0%]
tests/unit/test_behavior_flags.py ...                                                                                    [  1%]
tests/unit/test_compilation.py .......                                                                                   [  1%]
...
-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
======================================================== tests coverage ========================================================
_______________________________________ coverage: platform darwin, python 3.11.9-final-0 _______________________________________

Coverage XML written to file coverage.xml
========================================= 1514 passed, 8 skipped, 3 warnings in 35.09s =========================================
  py: OK (140.68=setup[92.42]+cmd[48.26] seconds)
  congratulations :) (140.70 seconds)
[WARNING] repo `https://github.com/pre-commit/pre-commit-hooks` uses deprecated stage names (commit, push) which will be removed in a future version.  Hint: often `pre-commit autoupdate --repo https://github.com/pre-commit/pre-commit-hooks` will fix this.  if it does not -- consider reporting an issue to that repo.
[WARNING] repo `https://github.com/pycqa/isort` uses deprecated stage names (commit, merge-commit, push) which will be removed in a future version.  Hint: often `pre-commit autoupdate --repo https://github.com/pycqa/isort` will fix this.  if it does not -- consider reporting an issue to that repo.
black................................................(no files to check)Skipped
flake8...............................................(no files to check)Skipped
mypy.................................................(no files to check)Skipped
```

### Troubleshooting

If you run into an error when you invoke `dbt` like:

```sh
dbt --version
```

```sh
Traceback (most recent call last):
  File "/Users/jeremy/git/dbt-core/venv/bin/dbt", line 3, in <module>
    from dbt.cli.main import cli
...
  File "/Users/jeremy/git/dbt-core/core/dbt/jsonschemas.py", line 14, in <module>
    from dbt.include.jsonschemas import JSONSCHEMAS_PATH
ModuleNotFoundError: No module named 'dbt.include.jsonschemas'
```

Then you need to set the `PYTHONPATH` env var to the `/core` directory where you've cloned the repo to:

```sh
# '/Users/jeremy/git' is where I cloned the dbt-core repo to.
export PYTHONPATH=/Users/jeremy/git/dbt-core/core
```

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
