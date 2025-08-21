---
---

# Indirect test selection

https://docs.getdbt.com/reference/node-selection/test-selection-examples?indirect-selection-mode=cautious#indirect-selection

More examples.

## Project setup

![alt text](image.png)

## Resulting DAG

![alt text](image-1.png)

## Eager mode (default)

```sh
$ dbt build -s b
$ dbt build -s b --indirect-selection=eager
```

![alt text](image-2.png)

## Buildable mode

```sh
$ dbt build -s b --indirect-selection=buildable
```

![alt text](image-3.png)

## Cautious mode

```sh
$ dbt build -s b --indirect-selection=cautious
```

![alt text](image-4.png)
