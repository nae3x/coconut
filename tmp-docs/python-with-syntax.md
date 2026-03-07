# Python `with` Statement (Context Managers)

## Basic Syntax

```python
with expression as variable:
    # block
```

The `with` statement evaluates `expression`, then uses the **result** as a context manager. The `as variable` part is optional.

---

## How It Works

The `with` statement relies on the **context manager protocol** — two special methods on an object:

| Method | When Called | Purpose |
|--------|-------------|---------|
| `__enter__(self)` | On entry | Setup; its return value is bound to the `as` variable |
| `__exit__(self, exc_type, exc_val, exc_tb)` | On exit (always) | Teardown; receives exception info if one occurred |

### Example: Custom Context Manager

```python
class MyContext:
    def __enter__(self):
        print("entering")
        return self  # bound to 'as' variable

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("exiting")
        return False  # False = do not suppress exceptions

with MyContext() as ctx:
    print("inside")

# Output:
# entering
# inside
# exiting
```

---

## Common Use Cases

### File Handling

```python
with open("file.txt") as f:
    data = f.read()
# file is always closed, even if an exception occurs
```

### Locks

```python
with lock:
    shared_resource.modify()
# lock is always released
```

### Multiple Context Managers (Python 3.1+)

```python
with open("a.txt") as a, open("b.txt") as b:
    ...
```

---

## Creating Context Managers with `@contextmanager`

The `contextlib.contextmanager` decorator lets you write a context manager as a generator function, without defining a class.

```python
from contextlib import contextmanager

@contextmanager
def managed():
    print("setup")
    try:
        yield "value"  # becomes the 'as' variable
    finally:
        print("teardown")

with managed() as v:
    print(v)  # "value"
```

- Code **before** `yield` acts as `__enter__`
- Code **after** `yield` (typically in `finally`) acts as `__exit__`
- The decorator returns a `_GeneratorContextManager` object that implements `__enter__` and `__exit__` for you

---

## Functions and `with`

`with` does **not** use a function itself as a context manager. It evaluates the expression and uses the **return value**:

```python
with some_function() as x:
    ...

# is equivalent to:
result = some_function()
with result as x:
    ...
```

So a function works with `with` only if it returns an object implementing the context manager protocol. `@contextmanager` is the standard way to make a function do this.

---

## Exception Handling in `__exit__`

`__exit__` receives three arguments when an exception occurs inside the block:

| Argument | Value |
|----------|-------|
| `exc_type` | The exception class |
| `exc_val` | The exception instance |
| `exc_tb` | The traceback object |

If no exception occurred, all three are `None`.

**Return value controls exception propagation:**

```python
def __exit__(self, exc_type, exc_val, exc_tb):
    if exc_type is ValueError:
        return True   # suppress the ValueError
    return False      # re-raise everything else (or None, same effect)
```

---

## What Happens Without the Protocol

If the object passed to `with` does not implement `__enter__` and `__exit__`, Python raises an `AttributeError` immediately:

```python
class Plain:
    pass

with Plain():
    pass

# AttributeError: __enter__
```

```python
def plain():
    return 42

with plain():  # 42 has no __enter__
    pass

# AttributeError: __enter__
```

There is no fallback — both methods are required.
