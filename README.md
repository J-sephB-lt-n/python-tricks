# python-tricks
Coding patterns that I'm trying to intentionally integrate into my vocabulary


# Contents 

**[Logging Decorator](#logging-decorator)**

**[Flatten a List of Lists](#flatten-a-list-of-lists)**

**[Function Caching](#function-caching)**

**[Group By and Summarize](#group-by-and-summarize)**

**[STILL IN PROGRESS: Sliding Window over an Iterator](#still-in-progress-sliding-window-over-an-iterator)**

# Logging Decorator
A logging decorator is a nice way to standardize the logging output format across a project, and also to make code shorter, more readable, more modular and more testable, by hiding the logging code behind a decorator abstraction. 

The logging decorator function shown below does the following:

* All logging is done via the python logger (in the standard library **logging** module). The logging decorator must be passed a python logger instance as an argument.

* The logging decorator automatically logs the function starting and finishing (it could easily be extended to also log the function input arguments).

* If the wrapped function raises an exception, then this exception is automatically logged.

* If the wrapped function returns a tuple of elements, where one (or more) of the elements are dictionaries with this format:

```python
# example logs returned by function #
{
    "logs":[
        (logging.INFO, {}),
        (logging.INFO, {}),
        (logging.DEBUG, {}),
        (logging.WARNING, {}),
        ...
    ]
}
```

...then each tuple ```(logging.<LEVEL>, {})``` contained within this list (i.e. within the *"logs"* key) will separately also be logged (after the function has finished). 

The format of the logs are standardised using a json-like **structured logging** format

The basic usage of the logging decorator looks like this:

```python
>>> import logging

>>> # set up a logger #
>>> logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
>>> logger = logging.getLogger()

>>> # function definition #
>>> @log_output(logger)     # this is the decorator which automatically logs at run-time (and automatically captures logs output by the function)
... def my_function(arg1, arg2, arg3):
...     logs_to_return = []
...     # do some function stuff here #
...     logs_to_return.append( 
...         (
...             logging.INFO,
...             StructLog(
...                 message="your custom message here",
...                 whatever_argname_you_want=69.420,
...                 another_arg_here="420",
...                 run_uuid="46290"
...             ),
...         ))
...     # do some more function stuff here
...     logs_to_return.append( 
...         (
...             logging.WARNING,
...             StructLog(
...                 message="some other message",
...                 run_uuid="46290"
...             ),
...         ))
...     func_result = "hello world"
...     return func_result, {"logs":logs_to_return}

# running the function #
>>> my_function(1,2,3)
INFO started running function my_function >>> {"datetime_utc": "2023-10-19 12:10:19"}
INFO your custom message here >>> {"whatever_argname_you_want": 69.42, "another_arg_here": "420", "run_uuid": "46290"}
WARNING some other message >>> {"run_uuid": "46290"}
INFO finished running function my_function >>> {"datetime_utc": "2023-10-19 12:10:19"}
'hello world'
# if the function raises an exception, then this is also automatically logged #
>>> my_function()
INFO started running function my_function >>> {"datetime_utc": "2023-10-19 12:10:47"}
ERROR Exception raised in my_function. exception: my_function() missing 3 required positional arguments: 'arg1', 'arg2', and 'arg3'
Traceback (most recent call last):
  File "<ipython-input-1-e90fe31c257e>", line 34, in func_wrapper
    func_results = func(*args, **kwargs)
                   ^^^^^^^^^^^^^^^^^^^^^
TypeError: my_function() missing 3 required positional arguments: 'arg1', 'arg2', and 'arg3'
```

Here is a slightly more complex working example:

```python
>>> # standard library imports #
>>> import datetime
>>> import logging
>>> import uuid 

>>> # 3rd party imports #
>>> import httpx

>>> # project-specific imports # 
>>> from logging_tools import log_output, StructLog

>>> # set up a logger # 
>>> logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
>>> logger = logging.getLogger()

# define our function #
>>> @log_output(logger)
... def scrape_webpage(url: str) -> str:
...     """(poorly written dummy example function) Scrapes [url], returning the response content as text"""
...     logs_list: list[dict] = []
...     scrape_id: str = uuid.uuid4().hex
...     logs_list.append(
...         (
...             logging.INFO,
...             StructLog(
...                 message="started scrape",
...                 scrape_id=scrape_id,
...                 step_name="pre_scrape",
...                 url=url,
...                 datetime_utc=datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S"),
...             ),
...         )
...     )
...     url_response: httpx.Response = httpx.get(url)
...     if url_response.status_code != 200:
...         # DONT DO THIS IN A REAL PROJECT: raise a specific exception #
...         raise Exception(f"received response {url_response.status_code}")
...     logs_list.append(
...         (
...             logging.INFO,
...             StructLog(
...                 message="finished scrape",
...                 response_len=len(url_response.text),
...                 scrape_id=scrape_id,
...                 step_name="post_scrape",
...                 url=url,
...                 datetime_utc=datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S"),
...             ),
...         )
...     )
...     return url_response.text, {"logs": logs_list}

>>> scrape_webpage("https://www.google.com")
INFO started running function scrape_webpage >>> {"datetime_utc": "2023-10-19 12:14:29"}
INFO HTTP Request: GET https://www.google.com "HTTP/1.1 200 OK"
INFO started scrape >>> {"scrape_id": "107bb9ea4ca342cab270ded0bf127056", "step_name": "pre_scrape", "url": "https://www.google.com", "datetime_utc": "2023-10-19 12:14:29"}
INFO finished scrape >>> {"response_len": 19518, "scrape_id": "107bb9ea4ca342cab270ded0bf127056", "step_name": "post_scrape", "url": "https://www.google.com", "datetime_utc": "2023-10-19 12:14:29"}
INFO finished running function scrape_webpage >>> {"datetime_utc": "2023-10-19 12:14:29"}
'<!doctype html><html itemscope="" ...'
>>> scrape_webpage("https://www.9gag.com")
INFO started running function scrape_webpage >>> {"datetime_utc": "2023-10-19 12:15:24"}
INFO HTTP Request: GET https://www.9gag.com "HTTP/1.1 301 Moved Permanently"
ERROR Exception raised in scrape_webpage. exception: received response 301
Traceback (most recent call last):
  File "<ipython-input-1-a3bb26fbdd67>", line 34, in func_wrapper
    func_results = func(*args, **kwargs)
                   ^^^^^^^^^^^^^^^^^^^^^
  File "<ipython-input-1-a3bb26fbdd67>", line 99, in scrape_webpage
    raise Exception(f"received response {url_response.status_code}")
Exception: received response 301
<exception stack trace follows>
```

Here is the definition of the decorator function and StructLog class:

```python
# logging_tools.py 

import datetime
import functools
import json
import logging

# define the structured logging class #
class StructLog(object):
    """
    A structured log (to be logged by python standard logging)
    
    refer to: https://docs.python.org/2/howto/logging-cookbook.html#implementing-structured-logging"""

    def __init__(self, message, **kwargs):
        self.message = message
        self.kwargs = kwargs

    def __str__(self):
        return "%s >>> %s" % (self.message, json.dumps(self.kwargs))

# define the logging decorator #
def log_output(logger: logging.Logger) -> None:
    """A decorator facilitating function logging"""
    def func_decorator(func): 
        @functools.wraps(func)
        def func_wrapper(*args, **kwargs):
            # log function start #
            logger.log(
                logger.level,
                StructLog(
                    message=f"started running function {func.__name__}",
                    datetime_utc=datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S"),
                )
            )
            # attempt to run function #
            try:
                func_results = func(*args, **kwargs)
            except Exception as err:
                logger.exception(f"Exception raised in {func.__name__}. exception: {str(err)}")
                raise err
            # if function returned additional logs, then log them #
            non_logging_func_results = []
            if isinstance(func_results, tuple):
                for func_result in func_results:
                    if isinstance(func_result, dict) and "logs" in func_result:
                        for log in func_result["logs"]:
                            logger.log(log[0], log[1])
                    else:
                        non_logging_func_results.append(func_result)
                if len(non_logging_func_results)==1:
                    non_logging_func_results = non_logging_func_results[0]
                else:
                    non_logging_func_results = tuple(non_logging_func_results)
            else:
                non_logging_func_results = func_results
            # log function completion #
            logger.log(
                logger.level,
                StructLog(
                    message=f"finished running function {func.__name__}",
                    datetime_utc=datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S"),
                )
            )
            return non_logging_func_results
        return func_wrapper
    return func_decorator
```



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

