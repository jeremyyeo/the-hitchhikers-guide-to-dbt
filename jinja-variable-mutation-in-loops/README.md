---
---

## Jinja variable mutation in loops

In Python, we can reassign a "global" variable in a loop:

```python
# m.py
v = 1
print(f'Value of v at start: {v}')
for i in [2, 3, 4]:
    v = i
    print(f'Value of v in for loop: {v}')
print(f'Value of v after for loop ends: {v}')
```

```sh
$ python m.py
Value of v at start: 1
Value of v in for loop: 2
Value of v in for loop: 3
Value of v in for loop: 4
Value of v after for loop ends: 4
```

You may try to apply your understanding above as you write dbt Jinja macros. But this doesn't work the same way due to "scoping" - https://jinja.palletsprojects.com/en/stable/templates/#assignments

```sql
{% macro m() %}
    {% set v = 1 %}
    {% do print('Value of v at start: ' ~ v) %}
    {% for i in [2, 3, 4] %}
        {% set v = i %}
        {% do print('Value of v in for loop: ' ~ v) %}
    {% endfor %}
    {% do print('Value of v after for loop ends: ' ~ v ~ '. Did you expect this to be 4?') %}
    {% do print('As we can see - Jinja loops are scoped only to the loop - so the outer v did not get set to 4 - even after the loop.') %}
    {% do print('Well, what can we do? We can use Jinja namespace.') %}

    {% set ns = namespace(vv=1) %}
    {% do print('Value of vv at start: ' ~ ns.vv) %}
    {% for i in [2, 3, 4] %}
        {% set ns.vv = i %}
        {% do print('Value of vv in for loop: ' ~ ns.vv) %}
    {% endfor %}
    {% do print('Value of vv after for loop ends: ' ~ ns.vv ~ '. We expected this to be 4 - and now it is so.') %}
{% endmacro %}
```

```sh
$ dbt run-operation m

09:17:27  Running with dbt=1.9.0-b4
09:17:27  Registered adapter: postgres=1.9.0-b1
09:17:27  Found 2 models, 546 macros
Value of v at start: 1
Value of v in for loop: 2
Value of v in for loop: 3
Value of v in for loop: 4
Value of v after for loop ends: 1. Did you expect this to be 4?
As we can see - Jinja loops are scoped only to the loop - so the outer v did not get set to 4 - even after the loop.
Well, what can we do? We can use Jinja namespace.
Value of vv at start: 1
Value of vv in for loop: 2
Value of vv in for loop: 3
Value of vv in for loop: 4
Value of vv after for loop ends: 4. We expected this to be 4 - and now it is so.
```
