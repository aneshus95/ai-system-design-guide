# Python Core Concepts

Python's language features come up constantly in coding screens for Data Science and ML roles — from how memory works with mutable types, to writing decorators and generators that make production code clean and efficient. Each concept below is explained in plain English with a small runnable example so you can build intuition fast.

## Table of Contents

- [Mutable vs Immutable Types](#mutable-vs-immutable-types)
- [`is` vs `==` and `id()`](#is-vs--and-id)
- [Variable Scope — LEGB, `global`, `nonlocal`](#variable-scope--legb-global-nonlocal)
- [`*args` and `**kwargs`](#args-and-kwargs)
- [Comprehensions](#comprehensions)
- [`lambda`, `map`, `filter`](#lambda-map-filter)
- [Closures](#closures)
- [Decorators](#decorators)
- [Generators and `yield`](#generators-and-yield)
- [Iterators — `__iter__` / `__next__`](#iterators----iter--next)
- [Context Managers](#context-managers)
- [Exception Handling](#exception-handling)
- [Shallow vs Deep Copy](#shallow-vs-deep-copy)
- [Type Hints](#type-hints)
- [The GIL — Threading vs Multiprocessing vs Asyncio](#the-gil--threading-vs-multiprocessing-vs-asyncio)
- [Interview Questions](#interview-questions)
- [References](#references)

---

## Mutable vs Immutable Types

**What it is:** Immutable objects cannot be changed after creation — any "change" creates a new object. Mutable objects can be modified in place. Think of an immutable object as a printed book (you'd need to print a new one to change it), and a mutable object as a whiteboard (erase and rewrite freely).

| Immutable | Mutable |
|-----------|---------|
| `int`, `float`, `bool` | `list` |
| `str`, `tuple` | `dict` |
| `frozenset` | `set` |

```python
# Immutable: reassignment makes a NEW object
x = "hello"
print(id(x))       # e.g. 140234567890
x = x + " world"
print(id(x))       # different id — new object

# Mutable: modification happens IN PLACE
nums = [1, 2, 3]
print(id(nums))    # e.g. 140234111111
nums.append(4)
print(id(nums))    # SAME id — same object, modified

# Classic gotcha: mutable default argument
def add_item(item, lst=[]):   # lst is shared across calls!
    lst.append(item)
    return lst

print(add_item(1))  # [1]
print(add_item(2))  # [1, 2]  ← surprise!

# Fix: use None as default
def add_item_safe(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

---

## `is` vs `==` and `id()`

**What it is:** `==` checks if two objects have the **same value**. `is` checks if they are the **exact same object in memory** (same address, returned by `id()`). Think of `==` as "do these two people have the same name?" and `is` as "are they literally the same person?"

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)   # True  — same value
print(a is b)   # False — different objects
print(a is c)   # True  — c points to the same object as a

print(id(a), id(b), id(c))  # a and c share an id; b is different

# Small int caching (CPython implementation detail)
x = 256
y = 256
print(x is y)   # True  — CPython caches small ints (-5 to 256)

x = 257
y = 257
print(x is y)   # False (usually) — outside cache range
```

---

## Variable Scope — LEGB, `global`, `nonlocal`

**What it is:** When Python looks up a variable name, it searches in this order — **L**ocal → **E**nclosing → **G**lobal → **B**uilt-in. It stops at the first match. `global` and `nonlocal` let you write to outer scopes instead of just reading them.

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        x = "local"
        print(x)   # "local"   (L)

    inner()
    print(x)       # "enclosing" (E)

outer()
print(x)           # "global"  (G)

# Modifying outer scopes
count = 0

def increment_global():
    global count
    count += 1

increment_global()
print(count)  # 1

def make_counter():
    n = 0
    def counter():
        nonlocal n    # modify enclosing n, not create a new local
        n += 1
        return n
    return counter

c = make_counter()
print(c(), c(), c())  # 1 2 3
```

---

## `*args` and `**kwargs`

**What it is:** `*args` collects any number of **positional** arguments into a tuple. `**kwargs` collects any number of **keyword** arguments into a dict. Think of `*args` as a catch-all bag for unnamed items, and `**kwargs` as a labelled bag for named items.

```python
def describe(*args, **kwargs):
    print("Positional:", args)
    print("Keyword:   ", kwargs)

describe(1, 2, 3, name="Alice", role="engineer")
# Positional: (1, 2, 3)
# Keyword:    {'name': 'Alice', 'role': 'engineer'}

# Unpacking into a function call
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
print(add(*nums))          # 6

params = {"a": 10, "b": 20, "c": 30}
print(add(**params))       # 60
```

---

## Comprehensions

**What it is:** A concise, readable way to build lists, dicts, or sets from iterables — often replacing a multi-line `for` loop with a single expression.

```python
# List comprehension
squares = [x**2 for x in range(10) if x % 2 == 0]
print(squares)  # [0, 4, 16, 36, 64]

# Dict comprehension
word_lengths = {word: len(word) for word in ["cat", "elephant", "ox"]}
print(word_lengths)  # {'cat': 3, 'elephant': 8, 'ox': 2}

# Set comprehension (deduplicates automatically)
unique_mods = {x % 3 for x in range(10)}
print(unique_mods)  # {0, 1, 2}

# Generator expression (lazy — see Generators section)
total = sum(x**2 for x in range(1_000_000))  # no giant list in memory
```

---

## `lambda`, `map`, `filter`

**What it is:** `lambda` creates a small anonymous function in one line — handy as a throwaway argument. `map` applies a function to every item in an iterable. `filter` keeps only items for which a function returns `True`.

```python
double = lambda x: x * 2
print(double(5))   # 10

nums = [1, 2, 3, 4, 5]

# map — apply a function to each element
doubled = list(map(lambda x: x * 2, nums))
print(doubled)     # [2, 4, 6, 8, 10]

# filter — keep elements where function returns True
evens = list(filter(lambda x: x % 2 == 0, nums))
print(evens)       # [2, 4]

# Modern preference: comprehensions are often clearer
doubled2 = [x * 2 for x in nums]
evens2   = [x for x in nums if x % 2 == 0]
```

---

## Closures

**What it is:** A closure is a function that "remembers" variables from its enclosing scope even after that outer function has finished running. Imagine a backpack that a function packs before leaving home — it carries those variables wherever it goes.

```python
def make_multiplier(factor):
    # 'factor' lives in the enclosing scope of the returned function
    def multiply(x):
        return x * factor   # 'factor' is closed over
    return multiply

triple = make_multiplier(3)
print(triple(10))   # 30  — 'factor=3' is still accessible

# Inspect what a closure has captured
print(triple.__closure__[0].cell_contents)  # 3
```

---

## Decorators

**What it is:** A decorator is a function that wraps another function to add behavior **without changing its code** — like gift-wrapping a box: the gift (original function) is unchanged, but now it has a bow (extra behavior).

### Basic decorator

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

@timer
def slow_sum(n):
    return sum(range(n))

slow_sum(1_000_000)  # slow_sum took 0.0341s
```

### Preserving metadata with `functools.wraps`

Without `@wraps`, the wrapper replaces the original function's `__name__` and `__doc__`, which breaks introspection and debugging.

```python
from functools import wraps

def timer(func):
    @wraps(func)   # copies __name__, __doc__, etc. from func to wrapper
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter()-start:.4f}s")
        return result
    return wrapper

@timer
def slow_sum(n):
    """Sum integers up to n."""
    return sum(range(n))

print(slow_sum.__name__)  # 'slow_sum'  (not 'wrapper')
print(slow_sum.__doc__)   # 'Sum integers up to n.'
```

### Decorator that takes arguments

Add one more layer of nesting to accept arguments before returning the actual decorator.

```python
from functools import wraps

def repeat(n_times):
    """Run the decorated function n_times."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(n_times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(3)
def say_hello(name):
    print(f"Hello, {name}!")

say_hello("Ada")
# Hello, Ada!
# Hello, Ada!
# Hello, Ada!
```

---

## Generators and `yield`

**What it is:** A generator is a function that produces values **one at a time** using `yield` instead of computing and storing them all at once. Think of it as a vending machine that dispenses one item per button press, rather than dumping everything on the floor at startup.

```python
import sys

# Regular list — all values in memory at once
def squares_list(n):
    return [x**2 for x in range(n)]

# Generator — one value at a time, on demand
def squares_gen(n):
    for x in range(n):
        yield x**2          # suspends here, resumes on next()

n = 1_000_000
lst = squares_list(n)
gen = squares_gen(n)

print(sys.getsizeof(lst))          # ~8 MB
print(sys.getsizeof(gen))          # ~112 bytes — constant regardless of n!

# Consume the generator
print(next(gen))    # 0
print(next(gen))    # 1

# Or iterate it
for val in squares_gen(5):
    print(val, end=" ")   # 0 1 4 9 16

# Generator expression (same idea, inline syntax)
gen_expr = (x**2 for x in range(5))
print(list(gen_expr))  # [0, 1, 4, 9, 16]
```

**Key gotcha:** A generator is exhausted after one full iteration — iterating it again yields nothing.

---

## Iterators — `__iter__` / `__next__`

**What it is:** An **iterable** is anything you can loop over (`list`, `str`, generator…). An **iterator** is an object that remembers its position and returns the next item on each `next()` call. Every iterator is an iterable, but not every iterable is an iterator.

```python
class Countdown:
    """An iterator that counts down from n to 1."""
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self          # the iterator IS itself

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        value = self.current
        self.current -= 1
        return value

for num in Countdown(5):
    print(num, end=" ")   # 5 4 3 2 1

# Under the hood, a for-loop does:
it = iter([10, 20, 30])  # calls __iter__
print(next(it))           # 10
print(next(it))           # 20
```

---

## Context Managers

**What it is:** A context manager wraps setup and teardown around a block of code, guaranteeing cleanup even if an exception occurs. The `with` statement is the most common use — it calls `__enter__` on the way in and `__exit__` on the way out, like an automatic door that always closes behind you.

### Class-based

```python
class ManagedFile:
    def __init__(self, path, mode):
        self.path, self.mode = path, mode

    def __enter__(self):
        self.file = open(self.path, self.mode)
        return self.file                  # value bound to 'as' variable

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()
        return False   # don't suppress exceptions

with ManagedFile("data.txt", "w") as f:
    f.write("hello")
# file is closed here, even if an exception was raised
```

### `contextlib.contextmanager` — simpler generator-based approach

```python
from contextlib import contextmanager

@contextmanager
def managed_file(path, mode):
    f = open(path, mode)
    try:
        yield f           # everything before yield = __enter__
    finally:
        f.close()         # everything after yield = __exit__

with managed_file("data.txt", "w") as f:
    f.write("hello")
```

---

## Exception Handling

**What it is:** Python's `try/except/else/finally` block lets you catch errors gracefully, run code only when no error occurred (`else`), and always run cleanup code (`finally`).

```python
# Full structure
try:
    result = 10 / int(input("Enter a number: "))
except ZeroDivisionError:
    print("Cannot divide by zero.")
except ValueError:
    print("Please enter a valid integer.")
else:
    print(f"Result: {result}")   # runs only if no exception
finally:
    print("Always runs — good for cleanup.")

# Custom exception
class ModelNotTrainedError(Exception):
    """Raised when predict() is called before fit()."""
    pass

def predict(model, X):
    if not model.is_fitted:
        raise ModelNotTrainedError("Call fit() before predict().")
    return model.transform(X)

# Catching and re-raising
try:
    predict(model, X)
except ModelNotTrainedError as e:
    print(f"Caught: {e}")
    raise   # re-raise the same exception up the call stack
```

---

## Shallow vs Deep Copy

**What it is:** A **shallow copy** creates a new container object but the nested objects inside still point to the same memory. A **deep copy** recursively copies everything — fully independent. Think of a shallow copy as photocopying a folder: you get a new folder, but the documents inside are still the originals.

```python
import copy

original = [[1, 2], [3, 4]]

shallow = copy.copy(original)      # or original[:]
deep    = copy.deepcopy(original)

original[0].append(99)

print(original)  # [[1, 2, 99], [3, 4]]
print(shallow)   # [[1, 2, 99], [3, 4]]  ← inner list is shared!
print(deep)      # [[1, 2], [3, 4]]      ← fully independent
```

---

## Type Hints

**What it is:** Type hints (PEP 484) let you annotate variables and function signatures with expected types. Python doesn't enforce them at runtime, but tools like `mypy` and IDEs use them to catch bugs early and improve autocomplete.

```python
from typing import Optional

def load_model(path: str, device: str = "cpu") -> Optional[object]:
    """Load a model from disk. Returns None if path doesn't exist."""
    ...

# Modern union syntax (Python 3.10+)
def process(data: list[float] | None) -> dict[str, float]:
    ...

# Variable annotations
weights: list[float] = []
learning_rate: float = 0.001

# Run mypy to check: mypy my_script.py
```

---

## The GIL — Threading vs Multiprocessing vs Asyncio

**What it is:** The **Global Interpreter Lock (GIL)** is a mutex in CPython that allows only **one thread to execute Python bytecode at a time**. It protects Python's internal reference-counting memory management but limits true CPU parallelism with threads.

**In plain English:** The GIL is a single-lane toll booth — no matter how many cars (threads) are queued, only one can pass at a time for Python code execution.

```
Task type          Best tool         Why
─────────────────────────────────────────────────────────────────
CPU-bound          multiprocessing   Each process has its own GIL
I/O-bound          threading         GIL released while waiting for I/O
Many async I/O     asyncio           Single thread, cooperative event loop
```

```python
import threading
import multiprocessing
import asyncio

# threading — good for I/O-bound (file reads, network calls)
def fetch(url):
    import urllib.request
    urllib.request.urlopen(url)   # GIL released during network wait

threads = [threading.Thread(target=fetch, args=("https://example.com",))
           for _ in range(5)]
for t in threads: t.start()
for t in threads: t.join()

# multiprocessing — good for CPU-bound (model training, image processing)
def crunch(n):
    return sum(x**2 for x in range(n))

with multiprocessing.Pool(4) as pool:
    results = pool.map(crunch, [10**6] * 4)

# asyncio — good for many concurrent I/O tasks (web servers, scrapers)
async def fetch_async(url, session):
    async with session.get(url) as resp:
        return await resp.text()

async def main():
    import aiohttp
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_async("https://example.com", session) for _ in range(5)]
        results = await asyncio.gather(*tasks)

# asyncio.run(main())
```

**Note:** Python 3.13 introduced an experimental "no-GIL" build (`--disable-gil`), but the GIL remains the default for now.

---

## Interview Questions

### Q: What is the difference between a mutable and immutable object? Give examples.
**Strong answer:**
Immutable objects (`int`, `str`, `tuple`) cannot be changed after creation; any operation that looks like a change actually creates a new object. Mutable objects (`list`, `dict`, `set`) can be modified in place. This matters for default arguments — using a mutable default like `def f(x=[])` is a classic bug because that list is shared across all calls.

---

### Q: What is the difference between `is` and `==`?
**Strong answer:**
`==` compares **values** (calls `__eq__`). `is` compares **identity** — whether two names point to the exact same object in memory, as returned by `id()`. Always use `==` for value comparison; `is` is mostly used to check `x is None` because `None` is a singleton.

---

### Q: Explain the LEGB rule.
**Strong answer:**
LEGB is the order Python searches for a name: **Local** (inside the current function) → **Enclosing** (outer functions, for nested functions) → **Global** (module level) → **Built-in** (Python builtins like `len`). Python stops at the first scope where it finds the name. `global` and `nonlocal` keywords let you write to outer scopes rather than just read from them.

---

### Q: What is a closure, and when would you use one?
**Strong answer:**
A closure is a nested function that captures and "closes over" variables from its enclosing scope, retaining them even after the outer function has returned. They're useful for factory functions (e.g., `make_multiplier(3)` returns a `triple` function) and for stateful callbacks without needing a class.

---

### Q: How do decorators work? Implement a simple timer decorator.
**Strong answer:**
A decorator is a callable that takes a function, wraps it in another function that adds behavior, and returns the wrapper. The `@` syntax is just sugar for `func = decorator(func)`. Always use `@functools.wraps(func)` inside a decorator to preserve the original function's `__name__` and `__doc__`.

```python
from functools import wraps
import time

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        t0 = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__}: {time.perf_counter()-t0:.4f}s")
        return result
    return wrapper
```

---

### Q: What is the difference between a generator and a list comprehension? When would you choose each?
**Strong answer:**
A list comprehension builds the entire list in memory at once (`[...]`). A generator expression (`(...)`) or a generator function (`yield`) produces values lazily, one at a time. Choose generators for large datasets, streaming data, or pipelines where you only need to iterate once — they use constant memory. Choose list comprehensions when you need random access, multiple iterations, or the data fits comfortably in memory.

---

### Q: What happens if you iterate over a generator twice?
**Strong answer:**
The second iteration yields nothing. Once a generator's `StopIteration` is raised, it is exhausted. You'd need to create a new generator object. This is a common gotcha when passing generators to functions — if the function iterates twice (e.g., first for length, then for values), the second pass is empty.

---

### Q: What is the GIL and how does it affect concurrency in Python?
**Strong answer:**
The Global Interpreter Lock (GIL) in CPython ensures only one thread executes Python bytecode at a time, protecting the reference-counting garbage collector. For **I/O-bound** tasks, threading still helps because the GIL is released while waiting on I/O. For **CPU-bound** tasks, use `multiprocessing` — separate processes each have their own GIL. `asyncio` avoids the GIL by running a single-threaded cooperative event loop, ideal for many concurrent I/O tasks.

---

### Q: What is the difference between `__iter__` and `__next__`?
**Strong answer:**
`__iter__` returns the iterator object itself (enabling use in `for` loops and `iter()` calls). `__next__` returns the next value and raises `StopIteration` when exhausted. An object that implements both is an **iterator**; an object that only implements `__iter__` (and returns a separate iterator) is merely an **iterable** (like a `list`).

---

### Q: What does `contextlib.contextmanager` do?
**Strong answer:**
It's a decorator that turns a generator function into a context manager, eliminating the need to write a full class with `__enter__` and `__exit__`. Code before `yield` runs on entry; code after `yield` (typically in a `finally` block) runs on exit — even if an exception is raised.

---

### Q: What is the difference between `copy.copy()` and `copy.deepcopy()`?
**Strong answer:**
`copy.copy()` creates a shallow copy — a new container, but nested objects inside still refer to the same memory. `copy.deepcopy()` recursively copies all nested objects, producing a fully independent clone. Use deep copy when you have nested mutable objects (e.g., lists of lists) and need changes to be completely isolated.

---

### Q: Why should you use `functools.wraps` inside a decorator?
**Strong answer:**
Without `@wraps(func)`, the wrapper function replaces the decorated function's `__name__`, `__doc__`, and other metadata with its own. This breaks `help()`, logging, stack traces, and testing tools that rely on function names. `@wraps(func)` copies those attributes from the original function to the wrapper automatically.

---

### Q: What are type hints and does Python enforce them?
**Strong answer:**
Type hints (PEP 484, added in Python 3.5) let you annotate function parameters and return values with expected types, e.g., `def fn(x: int) -> str`. Python does **not** enforce type hints at runtime — they are purely for documentation and static analysis. Tools like `mypy` and `pyright` perform static type checking, and IDEs use hints for autocomplete and early error detection.

---

### Q: When would you use `*args` vs `**kwargs`?
**Strong answer:**
Use `*args` when you want to accept a variable number of **positional** arguments (collected as a tuple). Use `**kwargs` when you want to accept a variable number of **keyword** arguments (collected as a dict). They're often combined in decorators — `wrapper(*args, **kwargs)` — so the wrapper can forward any arguments to the original function unchanged.

---

## References

- [Decorators in Python — GeeksforGeeks](https://www.geeksforgeeks.org/python/decorators-in-python/)
- [Python List Comprehensions vs Generator Expressions — GeeksforGeeks](https://www.geeksforgeeks.org/python/python-list-comprehensions-vs-generator-expressions/)
- [Global Interpreter Lock (GIL) in Python — DEV Community](https://dev.to/imsushant12/global-interpreter-lock-gil-in-python-everything-you-need-to-know-for-interviews-5e4g)
- [Python Scopes, Closures and the LEGB Rule — Medium](https://elfi-y.medium.com/python-scopes-closures-and-the-legb-rule-fc9d0f81705c)
- [Python Interview Questions 2025 — TechInterview.org](https://www.techinterview.org/post/3233474450/python-interview-questions-2025-generators-decorators-async-await-type-hints-dataclasses-concurrency-gil-memory-management/)

---

*Previous: [ML System Design](../20-machine-learning-foundations/04-ml-system-design.md) | Next: [OOP in Python](02-oop-in-python.md)*
