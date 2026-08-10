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

> **Why (the rationale):** Immutability guarantees that shared references can never be silently changed by another part of the code — critical for hash-based collections (`dict` keys, `set` members) and safe defaults in function signatures.
> **When to use:** Prefer immutable types (`tuple`, `frozenset`, `str`) when the data should never change after creation, when objects are used as dict keys, or when sharing across threads. Use mutable types (`list`, `dict`) when you need in-place growth or modification.
> **Nuances & gotchas:** Using a mutable default argument (`def f(x=[])`) is the single most common Python gotcha — the list is created once and shared across all calls; use `None` and initialise inside the function instead. Also, `tuple` is immutable but can contain mutable objects — reassigning an element raises `TypeError`, but mutating an inner list does not.

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

> **Why (the rationale):** Without the identity/equality distinction, comparing objects like `None` or singleton flags would be ambiguous — `is None` is faster and semantically precise because `None` is a singleton, while `==` delegates to `__eq__` which can be overridden.
> **When to use:** Use `is` only for `None`, `True`, `False`, and explicit singleton checks. Use `==` for all value comparisons.
> **Nuances & gotchas:** CPython interns small integers (-5 to 256) and many short strings, so `x is y` can return `True` for small ints even when you didn't intend identity comparison — this is a CPython implementation detail, not guaranteed by the language spec.

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

> **Why (the rationale):** Strict scope rules prevent accidental name collisions between functions — each function gets its own local namespace, so local variables don't bleed into callers. `global`/`nonlocal` provide controlled escape hatches when you genuinely need to mutate state in an outer scope.
> **When to use:** Use `global` sparingly for module-level configuration or counters. Use `nonlocal` inside nested functions (e.g., closures that need to update a counter). Prefer returning values or using a class when `global` usage grows.
> **Nuances & gotchas:** If you assign to a name inside a function at any point, Python treats it as local for the entire function — reading it before the assignment raises `UnboundLocalError`, even if a global with the same name exists. This surprises many beginners.

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

> **Why (the rationale):** They make APIs flexible without breaking callers when you add new parameters — decorators use `wrapper(*args, **kwargs)` to forward all arguments to the original function unchanged.
> **When to use:** Use `*args` when the number of positional inputs is genuinely variable (e.g., `sum`, `print`). Use `**kwargs` for option-passing APIs and decorator wrappers. Avoid overusing both — explicit parameters are clearer and support better tooling (autocomplete, type checking).
> **Nuances & gotchas:** Argument order in a signature must follow: positional-only → `*args` → keyword-only → `**kwargs`. Passing keyword arguments as positional (or vice versa) raises `TypeError`. In Python 3, you can place `*` alone in the signature to force all subsequent arguments to be keyword-only.

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

> **Why (the rationale):** Comprehensions are typically faster than equivalent `for`-loop + `append` code (the loop body executes as a C-level iteration rather than repeated Python function calls) and they express intent in one readable line.
> **When to use:** Use list comprehensions when you need the full collection immediately (random access, `len`, multiple passes). Switch to a generator expression (parentheses) when processing large data you'll only iterate once — it keeps memory at O(1) instead of O(n).
> **Nuances & gotchas:** Comprehensions have their own scope in Python 3 (unlike Python 2), so the loop variable doesn't leak. Nested comprehensions `[f(x) for row in matrix for x in row]` execute left-to-right, which often reads backwards compared to the nested `for` loop equivalent — always double-check the order.

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

> **Why (the rationale):** `lambda` avoids the overhead of a named `def` when you only need a trivial function in one place (e.g., a `key=` argument to `sorted`). `map` and `filter` return lazy iterators, so they avoid building intermediate lists.
> **When to use:** Use `lambda` for simple one-expression callbacks (`sorted(items, key=lambda x: x.name)`). For anything requiring multiple lines or a name, use a full `def`. In modern Python, prefer list/generator comprehensions over `map`/`filter` — they are more readable and support arbitrary expressions without a lambda.
> **Nuances & gotchas:** `lambda` cannot contain statements (assignments, `return`, `for`). `map` and `filter` return iterators in Python 3 — wrap in `list()` if you need random access. Chaining `map(map(…))` creates deeply nested iterators that are hard to debug.

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

> **Why (the rationale):** Closures give you lightweight stateful behaviour without needing a class — a factory function returns a pre-configured inner function that carries its configuration in a cell variable.
> **When to use:** Use closures for factory functions (`make_multiplier`, `make_validator`) and simple stateful callbacks. Prefer a class when the state grows complex (multiple variables, methods) or when you need pickling/serialization.
> **Nuances & gotchas:** All inner functions in a loop share the same cell variable for the loop index — `lambda: i` in a loop will capture the final value of `i`, not the value at the time of creation. Fix with a default-argument trick: `lambda i=i: i`.

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

> **Why (the rationale):** Decorators implement cross-cutting concerns (logging, timing, auth, caching, retry logic) in one place that is reused across many functions — keeping the decorated functions clean and single-purpose.
> **When to use:** Use decorators for behaviour that is orthogonal to the function's core logic and needed across multiple functions. Avoid them for logic that only applies to one function — a plain helper call is simpler.
> **Nuances & gotchas:** Always add `@functools.wraps(func)` inside the decorator — without it the wrapper replaces `__name__`, `__doc__`, and signature, breaking `help()`, stack traces, and testing tools. Stacking multiple decorators applies them bottom-up: `@A @B def f` means `f = A(B(f))`.

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

> **Why (the rationale):** Generators use constant memory regardless of dataset size — a generator over 1 million rows occupies ~112 bytes versus ~8 MB for the equivalent list — making them essential for streaming data, large files, and ML data pipelines.
> **When to use:** Use generators for large sequences you only need to iterate once (file processing, streaming API responses, infinite sequences). Use lists when you need random access, `len()`, multiple iterations, or slicing.
> **Nuances & gotchas:** Generators are lazy and single-pass — once exhausted, re-iterating yields nothing (no error, just empty). You cannot index (`gen[0]`) or get `len()` of a generator. If a function receives a generator and iterates it twice (e.g., first for count, then for values), the second pass will be empty.

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

> **Why (the rationale):** The iterator protocol provides a universal contract for sequential data access — `for` loops, `zip`, `enumerate`, and comprehensions all rely on it, so any object implementing `__iter__`/`__next__` integrates seamlessly with Python's ecosystem.
> **When to use:** Implement a custom iterator class when you need lazy, stateful traversal with complex logic (e.g., sliding windows, tree traversal) and want full control over state. For simple cases, a generator function is shorter and preferred.
> **Nuances & gotchas:** A `list` is iterable but NOT an iterator — calling `next()` on a list raises `TypeError`. You must first call `iter(my_list)` to get an iterator. Forgetting this is a common bug when manually driving iteration. Iterators returned by built-ins (`map`, `filter`, `zip`) are also single-pass.

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

> **Why (the rationale):** `try/finally` blocks ensure cleanup but are verbose and easy to forget. Context managers encapsulate the acquire-release pattern in a reusable, composable object so callers can't accidentally skip teardown.
> **When to use:** Use context managers for any resource that needs deterministic release: files, network connections, database sessions, locks, temporary directories, and timers. Prefer `@contextlib.contextmanager` for one-off cases; write a full class only when you need the context manager to be reusable with complex state.
> **Nuances & gotchas:** `__exit__` receives the exception info (`exc_type`, `exc_val`, `exc_tb`). Returning `True` from `__exit__` silently suppresses the exception — only do this intentionally. With `@contextmanager`, the `yield` must be inside a `try/finally` so the `finally` always runs; omitting it means exceptions inside the `with` block skip the cleanup code.

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

> **Why (the rationale):** Exceptions separate error-handling from normal logic, and `finally` guarantees cleanup even during unexpected failures — making code more robust than checking return codes at every step.
> **When to use:** Catch the most specific exception type possible, not bare `except:` which silences everything including `KeyboardInterrupt` and `SystemExit`. Use `else` for code that should only run when no exception occurred (cleaner than nesting it in `try`). Reserve custom exceptions for domain-specific error conditions that callers may want to handle differently.
> **Nuances & gotchas:** Catching `Exception` still lets `SystemExit`, `KeyboardInterrupt`, and `GeneratorExit` propagate (they inherit from `BaseException`). Re-raising with bare `raise` preserves the original traceback; re-raising with `raise e` loses it. Using `except Exception as e: pass` silently swallows all errors — add at least a log statement.

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

> **Why (the rationale):** Without explicit copying, assignment (`b = a`) just creates another reference to the same object — mutations via either name affect the same data. Copying creates independent objects so modifications to one don't bleed into the other.
> **When to use:** Use shallow copy when the container itself is new but you intentionally want to share the inner objects (or they are immutable). Use deep copy when nested objects are mutable and must be fully independent (e.g., copying a list of lists for separate processing).
> **Nuances & gotchas:** Deep copy is significantly slower and more memory-intensive for large nested structures. It also copies objects that may not be meant to be copied (open file handles, sockets, locks) — `deepcopy` can fail or produce broken objects for these types. Some classes override `__copy__`/`__deepcopy__` to control copying behaviour.

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

> **Why (the rationale):** Type hints catch whole categories of bugs (wrong argument type, missing return) without running the code, and they serve as inline documentation that IDEs and linters can verify, reducing review burden.
> **When to use:** Add type hints to all public function signatures in production code and shared libraries. They are especially valuable in large codebases and when working in teams. For small scripts or rapid prototyping, they are optional overhead.
> **Nuances & gotchas:** Python does NOT enforce type hints at runtime — passing the wrong type raises no error unless you add a runtime checker (`beartype`, `pydantic`). `Optional[X]` is equivalent to `X | None` (Python 3.10+). Annotating with `list` vs `List` (from `typing`) was a Python 3.9 change — use the lowercase built-in forms in 3.9+ for simplicity.

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

> **Why (the rationale):** The GIL was a pragmatic design choice to simplify CPython's reference-counting garbage collector and make single-threaded code faster — but it means Python threads don't parallelize CPU-bound work.
> **When to use:** Use `threading` for I/O-bound concurrency (network calls, file reads) where the GIL is released during I/O waits. Use `multiprocessing` for CPU-bound parallelism (model training, image processing) — separate processes have separate GILs. Use `asyncio` for high-concurrency I/O (hundreds of simultaneous connections) with low overhead on a single thread.
> **Nuances & gotchas:** Threads don't help for CPU-bound work — they may be *slower* than single-threaded due to lock contention. `multiprocessing` incurs process-startup and inter-process serialization (pickle) overhead — avoid passing large arrays between processes (use shared memory or `numpy` with shared arrays instead). `asyncio` requires all blocking calls to be explicitly awaited — mixing synchronous blocking calls into an async event loop freezes all other coroutines.

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

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Immutable Object** | An object whose value cannot be changed after creation; any modification creates a new object | Guarantees that values cannot be accidentally altered by other parts of the code |
| **Mutable Object** | An object that can be modified in place without creating a new one | Enables efficient in-place updates; requires care to avoid unintended shared-state bugs |
| **`id()` Function** | Returns the unique integer memory address of an object in CPython | Used with `is` to check whether two variable names refer to the exact same object |
| **`is` Operator** | Tests whether two names point to the same object in memory (identity check) | Appropriate only for checking `None` or singleton comparisons; use `==` for value equality |
| **`==` Operator** | Tests whether two objects have the same value by calling `__eq__` | The correct way to compare values; distinct from identity checking with `is` |
| **LEGB Rule** | The order Python searches for a variable name: Local → Enclosing → Global → Built-in | Determines which variable a name refers to in any given scope |
| **`global` Keyword** | Declares that a variable assignment inside a function refers to the module-level global variable | Allows functions to read and modify module-level state instead of creating a local shadow |
| **`nonlocal` Keyword** | Declares that a variable assignment refers to the nearest enclosing (non-global) scope | Enables nested functions to modify a variable defined in an outer function |
| **`*args`** | Syntax that collects any number of extra positional arguments into a tuple inside a function | Makes functions accept a flexible number of positional inputs |
| **`**kwargs`** | Syntax that collects any number of extra keyword arguments into a dictionary inside a function | Makes functions accept a flexible number of named inputs |
| **List Comprehension** | A concise one-line expression that builds a new list by iterating and optionally filtering an iterable | More readable and typically faster than an equivalent for-loop when building lists |
| **Dict Comprehension** | A concise expression that builds a dictionary from an iterable in a single line | Replaces verbose loop-based dictionary construction with readable declarative syntax |
| **Generator Expression** | A lazy comprehension using parentheses that produces values one at a time instead of building a full list | Constant memory usage regardless of input size; ideal for large or streaming data |
| **`lambda` Function** | An anonymous single-expression function defined inline | Used as throwaway callbacks in `map`, `filter`, `sorted`, and similar higher-order functions |
| **`map()` Function** | Applies a function to every element of an iterable and returns an iterator of results | Transforms each element without writing an explicit for-loop |
| **`filter()` Function** | Returns an iterator containing only elements of an iterable for which a function returns True | Selects a subset of elements without an explicit loop |
| **Closure** | A nested function that remembers and can access variables from its enclosing scope after the outer function has returned | Used to create stateful function factories without requiring a full class |
| **Decorator** | A function that wraps another function to add behaviour without modifying the original function's code | Enables cross-cutting concerns like logging, timing, authentication, and caching to be applied cleanly |
| **`functools.wraps`** | A decorator that copies the wrapped function's `__name__`, `__doc__`, and other metadata onto the wrapper | Preserves introspection, logging, and debugging information when applying decorators |
| **Generator Function** | A function that uses `yield` to produce values lazily, one at a time, suspending execution between calls | Enables memory-efficient processing of large datasets or infinite sequences |
| **`yield` Keyword** | Pauses a generator function and sends a value to the caller; resumes from the same point on the next call | The mechanism that makes generator functions lazy; replaces return in sequential data-producing functions |
| **Iterator** | An object that implements `__iter__` and `__next__`, maintaining position and returning the next item on demand | The underlying protocol that enables for-loops and `next()` calls on any data source |
| **Iterable** | Any object that implements `__iter__` and returns an iterator | The broader category; lists, strings, and generators are all iterables |
| **`StopIteration`** | The exception an iterator raises when it has no more items to yield | The signal that a for-loop or `next()` call uses to know when iteration is complete |
| **Context Manager** | An object that defines setup (`__enter__`) and teardown (`__exit__`) logic for use with the `with` statement | Guarantees cleanup code runs even if an exception occurs inside the `with` block |
| **`with` Statement** | A block that automatically calls `__enter__` on entry and `__exit__` on exit of a context manager | The standard way to handle resources like files, locks, and database connections safely |
| **`contextlib.contextmanager`** | A decorator that turns a generator function into a context manager | Avoids writing a full `__enter__`/`__exit__` class for simple resource-management patterns |
| **`try / except / else / finally`** | Python's exception handling structure for catching errors, running code on success, and always running cleanup | Enables robust error handling and guaranteed resource cleanup |
| **Custom Exception** | A user-defined exception class inheriting from `Exception` for domain-specific error conditions | Makes error types descriptive and allows callers to catch specific error categories |
| **Shallow Copy** | A copy that creates a new container object but leaves nested objects pointing to the same memory | Faster than deep copy; sufficient when nested objects will not be mutated |
| **Deep Copy** | A recursive copy that duplicates all nested objects, producing a fully independent clone | Required when nested mutable objects must be modified without affecting the original |
| **Type Hints (PEP 484)** | Optional annotations on function parameters and return values declaring expected types | Enables static analysis tools and IDEs to catch type errors without running the code |
| **`mypy`** | A static type-checker that reads Python type hints and reports type errors before runtime | Catches classes of bugs (wrong argument types, missing returns) that tests might miss |
| **GIL (Global Interpreter Lock)** | A mutex in CPython that allows only one thread to execute Python bytecode at a time | Simplifies CPython's memory management but prevents true CPU parallelism with threads |
| **`threading`** | Python's module for running multiple threads in the same process | Effective for I/O-bound tasks because the GIL is released while waiting on I/O |
| **`multiprocessing`** | Python's module for spawning separate processes, each with its own Python interpreter and GIL | The correct approach for CPU-bound parallelism; avoids the GIL entirely |
| **`asyncio`** | Python's cooperative concurrency framework using a single-threaded event loop and `async/await` | Efficient for many concurrent I/O tasks without the overhead of creating threads or processes |
| **CPU-Bound Task** | A task limited by processor speed, such as numerical computation or model training | Requires `multiprocessing` to achieve parallelism in Python due to the GIL |
| **I/O-Bound Task** | A task limited by waiting on external resources such as network, disk, or databases | Can benefit from threading or asyncio since the GIL is released during I/O waits |

*Previous: [ML System Design](../20-machine-learning-foundations/04-ml-system-design.md) | Next: [OOP in Python](02-oop-in-python.md)*
