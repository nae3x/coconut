# Python `*args` and `**kwargs` — Detailed Explanation

## Overview

Python provides two special syntaxes for functions to accept a variable number of arguments:

- `*args` — collects extra **positional** arguments into a `tuple`
- `**kwargs` — collects extra **keyword** arguments into a `dict`

These are conventions; the names `args` and `kwargs` are not required, but are universally used.

---

## `*args` — Variable Positional Arguments

When you prefix a parameter with `*`, the function accepts any number of positional arguments beyond the explicitly named ones. They are bundled into a tuple.

```python
def greet(*args):
    for name in args:
        print(f"Hello, {name}!")

greet("Alice", "Bob", "Charlie")
# Hello, Alice!
# Hello, Bob!
# Hello, Charlie!
```

### With required positional parameters

```python
def foo(first, second, *rest):
    print(first)   # required
    print(second)  # required
    print(rest)    # tuple of the rest

foo(1, 2, 3, 4, 5)
# 1
# 2
# (3, 4, 5)
```

### Key properties

- Type inside the function: `tuple`
- Order matters — arguments are collected left-to-right
- Can be empty: `foo()` gives `args = ()`

---

## `**kwargs` — Variable Keyword Arguments

When you prefix a parameter with `**`, the function accepts any number of keyword arguments not matched by other parameters. They are bundled into a dict.

```python
def describe(**kwargs):
    for key, value in kwargs.items():
        print(f"{key} = {value}")

describe(name="Alice", age=30, city="Amsterdam")
# name = Alice
# age = 30
# city = Amsterdam
```

### With required keyword parameters

```python
def connect(host, port, **options):
    print(host, port)
    print(options)

connect("localhost", 5432, timeout=10, ssl=True)
# localhost 5432
# {'timeout': 10, 'ssl': True}
```

### Key properties

- Type inside the function: `dict`
- Order is preserved (Python 3.7+)
- Can be empty: `foo()` gives `kwargs = {}`

---

## Parameter Ordering Rules

Python enforces a strict order for parameter kinds in a function signature:

```
def foo(positional_only, /, normal, *args, keyword_only, **kwargs):
```

| Position | Kind | Separator |
|---|---|---|
| Before `/` | Positional-only | `/` (Python 3.8+) |
| Between `/` and `*` | Normal (either way) | — |
| After `*` or `*args` | Keyword-only | `*` |
| Last | `**kwargs` | — |

Violating this order is a `SyntaxError`.

---

## Positional-only Parameters (`/`)

Introduced in Python 3.8. Parameters before `/` **cannot** be passed by name.

```python
def foo(a, b, /, c):
    pass

foo(1, 2, 3)       # OK
foo(1, 2, c=3)     # OK
foo(a=1, b=2, c=3) # TypeError: a and b are positional-only
```

Useful for:
- Matching C extension function signatures
- Allowing the parameter name to change without breaking callers

---

## Keyword-only Parameters (`*`)

Parameters after `*` (or after `*args`) **must** be passed by name.

```python
def foo(a, b, *, c, d=10):
    pass

foo(1, 2, c=3)        # OK
foo(1, 2, c=3, d=20)  # OK
foo(1, 2, 3)          # TypeError: too many positional arguments
```

Useful for:
- Forcing callers to be explicit, improving readability
- Adding options to a function without breaking existing positional callers

---

## Using All Together

```python
def foo(pos_only, /, normal, *args, kw_only, **kwargs):
    print("pos_only:", pos_only)
    print("normal:  ", normal)
    print("args:    ", args)
    print("kw_only: ", kw_only)
    print("kwargs:  ", kwargs)

foo(1, 2, 3, 4, kw_only=5, extra=6)
# pos_only: 1
# normal:   2
# args:     (3, 4)
# kw_only:  5
# kwargs:   {'extra': 6}
```

---

## Unpacking at the Call Site

`*` and `**` can also be used when **calling** a function to unpack sequences and dicts.

```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
add(*nums)          # same as add(1, 2, 3)

opts = {"a": 1, "b": 2, "c": 3}
add(**opts)         # same as add(a=1, b=2, c=3)
```

### Combining unpacking

```python
def foo(a, b, c, d):
    pass

positional = (1, 2)
keyword = {"c": 3, "d": 4}

foo(*positional, **keyword)  # foo(1, 2, c=3, d=4)
```

---

## Common Patterns

### Forwarding all arguments

```python
def wrapper(*args, **kwargs):
    return original(*args, **kwargs)
```

### Mixing fixed and variable args

```python
def log(level, *messages, sep=" "):
    print(f"[{level}]", sep.join(messages))

log("INFO", "Starting", "server", sep="-")
# [INFO] Starting-server
```

### Enforcing keyword-only for clarity

```python
def create_user(name, *, email, role="user"):
    # email must always be named — prevents mix-ups
    pass

create_user("Alice", email="alice@example.com")
```

---

## Summary Table

| Syntax | Collects | Type | Passed as |
|---|---|---|---|
| `*args` | Extra positional args | `tuple` | `foo(1, 2, 3)` |
| `**kwargs` | Extra keyword args | `dict` | `foo(a=1, b=2)` |
| `/` boundary | Positional-only params | — | must be positional |
| `*` boundary | Keyword-only params | — | must be named |
