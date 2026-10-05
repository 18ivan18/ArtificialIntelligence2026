# Python Intro

A recap of the Python we rely on for the rest of the course, before any AI shows up.
Inspired by Stanford's CS224n Python tutorial.

## Contents

- **Collections** — lists and slicing, tuples, dictionaries
- **Sets** — uniqueness, O(1) membership, hashability
- **collections** — `defaultdict` for adjacency lists, `Counter` for tallies
- **heapq** — the priority queue, and why `(priority, item)` tuples break on ties
- **Loops** — iterating over lists and dictionaries, list comprehensions
- **Functions** — default arguments, type hints, `*args` & `**kwargs`
- **Higher-order functions** — `map`, `filter`, `zip`, lambdas
- **Sorting with `key=`** — `sorted`, `min` and `max` with a key function
- **Closures** — free variables, and the late-binding trap in `create_adders`
- **Decorators** — writing one by hand, then `functools.wraps` and `lru_cache`
- **OOP** — classes, exceptions, duck typing, dunder methods
- **Dataclasses** — generated `__init__`/`__repr__`/`__eq__`, `field(default_factory=...)`,
  and `frozen=True, order=True` for states that go in a heap or a set
- **The typing module** — collection generics, `Any`, `Callable`, `Optional`
- **NumPy** — arrays, axis reductions, matrix products, indexing, broadcasting, and why explicit
  for-loops over arrays are to be avoided

## Why this comes first

Almost everything later in the course is either a search over states you keep in Python
collections, or a NumPy array you must not loop over.
