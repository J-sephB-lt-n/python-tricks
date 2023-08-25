# python-tricks
Coding patterns that I'm trying to intentionally integrate into my vocabulary


# Contents 

**[Flatten a List of Lists](#flatten-a-list-of-lists)**

**[Function Caching](#function-caching)**

**[Group By and Summarize](#group-by-and-summarize)**

**[STILL IN PROGRESS: Sliding Window over an Iterator](#still-in-progress-sliding-window-over-an-iterator)**

# Flatten a List of Lists
```python
>>> import itertools
>>> list_of_lists: list[list[int]] = [[1,2],[3,4,5],[6],[7,8,9,10]]
>>> list( itertools.chain(*list_of_lists) )
[1,2,3,4,5,6,7,8,9,10]
```
## Function Caching

```python
import functools
import time

def time_function(func):
    """Logs time taken to run a function
    Based this function on code from: 
    https://stackoverflow.com/questions/1622943/timeit-versus-timing-decorator"""

    @functools.wraps(func)
    def wrap(*args, **kwargs):
        time_start = time.perf_counter()
        result = func(*args, **kwargs)
        time_end = time.perf_counter()
        print(
            "func:%r args:[%r, %r] took: %2.4f sec"
            % (func.__name__, args, kwargs, time_end - time_start)
        )
        return result

    return wrap

@time_function
@functools.cache # remembers everything (inifinite memory)
def number_squared(num: int) -> None:
    time.sleep(num)
    return num * num

print(number_squared(1))  # took 1 sec
print(number_squared(2))  # took 2 secs
print(number_squared(3))  # took 3 secs
print(number_squared(1))  # took <0 secs
print(number_squared(2))  # took <0 secs
print(number_squared(3))  # took <0 secs

@time_function
@functools.lru_cache(maxsize=3)  # same as cache but has a limited memory
def number_squared_memory3(num: int) -> None:
    time.sleep(num)
    return num * num

print(number_squared_memory3(1))  # took 1 sec
print(number_squared_memory3(2))  # took 2 secs (remembers 1)
print(number_squared_memory3(3))  # took 3 secs (remembers 1,2)
print(number_squared_memory3(4))  # took 4 secs (remembers 1,2,3)
print(number_squared_memory3(2))  # took <0 secs (remembers 2,3,4)
print(number_squared_memory3(3))  # took <0 secs (remembers 2,3,4)
print(number_squared_memory3(4))  # took <0 secs (remembers 2,3,4)
print(number_squared_memory3(1))  # took 1 sec (remembers 2,3,4)
print(number_squared_memory3(3))  # took <0 secs (remembers 3,4,1)
print(number_squared_memory3(2))  # took 2 secs (remembers 3,4,1)
```

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

## STILL IN PROGRESS: Sliding Window over an Iterator

!!I'm still checking this code out!!

```python
import itertools
from typing import Any, Iterator

def window(seq, window_len:int):
    "Returns a sliding window (of width n) over data from the iterable"
    "   s -> (s0,s1,...s[n-1]), (s1,s2,...,sn), ...                   "
    it = iter(seq)
    result = tuple(itertools.islice(it, window_len))
    if len(result) == window_len:
        yield result
    for elem in it:
        result = result[1:] + (elem,)
        yield result
    
def list_to_generator(values_list: list[Any]) -> Iterator[int]:
    """Converts a list into a generator"""
    for value in values_list:
        yield value

nums = list_to_generator([23, -68, 5, -42, 48, 63, 93, -57, -96, 44])
```

