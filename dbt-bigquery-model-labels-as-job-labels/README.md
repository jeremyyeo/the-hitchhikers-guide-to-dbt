---
---

# Using model labels as BigQuery job labels

```yaml
# dbt_project.yml
name: analytics
profile: bq
version: "1.0.0"

models:
  analytics:
    +materialized: table

query-comment:
  comment: "{{ query_comment(node) }}"
  job-label: true
```

```sql
-- macros/query_comment.sql
{% macro query_comment(node) %}
    {%- set comment_dict = {} -%}
    {%- do comment_dict.update(
        app='dbt',
        dbt_version=dbt_version,
        profile_name=target.get('profile_name'),
        target_name=target.get('target_name'),
    ) -%}
    {%- if node is not none -%}
      {%- do comment_dict.update(node.config.get("labels")) -%}
    {% else %}
      {%- do comment_dict.update(node_id='internal') -%}
    {%- endif -%}
    {% do return(tojson(comment_dict)) %}
{% endmacro %}

-- models/foo.sql
{{ config(labels = {'my_label_a': 'foo', 'my_label_b': 'foo'}) }}
select 1 c

-- models/bar.sql
{{ config(labels = {'my_label_c': 'bar'}) }}
select 2 c
```

Both the object and the job that built those objects will have the model's labels:

![alt text](image.png)

![alt text](image-1.png)
