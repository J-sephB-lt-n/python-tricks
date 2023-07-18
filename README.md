# python-tricks
Coding patterns that I'm trying to intentionally integrate into my vocabulary

## Group By and Summarize

```python
>>> import itertools
>>> import statistics
>>> from typing import Iterator

>>> users_db: list[dict[str, int]] = [
...     {"user_id": 0, "gender": 1, "age": 12},
...     {"user_id": 1, "gender": 0, "age": 69},
...     {"user_id": 2, "gender": 1, "age": 4},
...     {"user_id": 3, "gender": 1, "age": 20},
...     {"user_id": 4, "gender": 0, "age": 55},
... ]

# for SQL-like grouping, itertools.groupby requires list sorted by grouping column #
>>> users_db = sorted(users_db, key=lambda x: x["gender"])

# count of users by gender #
>>> gender_grouped_data: Iterator[tuple[int, Iterator[dict[str, int]]]] = itertools.groupby(
...    iterable=users_db, 
...    key=lambda x: x["gender"],
... )
>>> for grp in gender_grouped_data:
...    print(f"gender={grp[0]}, n_users={sum(1 for _ in grp[1])}")
gender=0, n_users=2
gender=1, n_users=3

# average age of users grouped by gender #
>>> gender_grouped_data = itertools.groupby(
...    iterable=users_db, 
...    key=lambda x: x["gender"]
... )
>>> for grp in gender_grouped_data:
...    print(
...        f'gender={grp[0]}, average_age={statistics.fmean([x["age"] for x in grp[1]])}'
... )
gender=0, average_age=62.0
gender=1, average_age=12.0
```