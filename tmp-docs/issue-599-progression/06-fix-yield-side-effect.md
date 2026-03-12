# Issue 599: Fix `keyword_funcdef_handle` Side Effect

## Problem

`keyword_funcdef_handle` (compiler.py) appends `if False: yield` to the **end** of the function body when processing `yield def`. When the body already ends with a terminator (e.g., `return x` from a math-def like `yield def f(x) = x`), the unreachable code checker flags the appended block as unreachable — a false positive since the compiler generated it, not the user.

## Plan

Change `keyword_funcdef_handle` to **prepend** the `if False: yield` block at the **start** of the function body instead of appending it at the end.

### Before (append)

```python
if kwd == "yield":
    funcdef += handle_indentation("""
if False:
    yield
    """, add_newline=True, extra_indent=1)
```

### After (prepend via insert after first `openindent`)

```python
if kwd == "yield":
    if_false_yield = handle_indentation("""
if False:
    yield
    """, add_newline=True)
    idx = funcdef.index(openindent) + 1
    funcdef = funcdef[:idx] + if_false_yield + funcdef[idx:]
```

### Why this works

- `funcdef` is a string like `def f(x):\n⁋<body>⁎` where `⁋` = `openindent` and `⁎` = `closeindent`.
- `funcdef.index(openindent) + 1` finds the first `openindent` (the function body start) and moves past it.
- The inserted block sits at the top of the body, before any `return` statements.
- `extra_indent` drops from 1 to 0 because the insertion point is already inside the body's indent level.

## Edge Cases Considered

### Functions with docstrings

```coconut
yield def f(x):
    """This is a docstring."""
    return x
```

After the fix, the compiled body becomes:

```python
def f(x):
    if False:
        yield
    """This is a docstring."""
    return x
```

The string literal is no longer recognized as a PEP 257 docstring by tooling, but this has no runtime impact. For match funcdefs, the docstring is parsed separately before the suite, so the `openindent` still marks the body start correctly.

### `copyclosure` + `yield` together

```coconut
copyclosure yield def f(x):
    return x
```

`copyclosure` prepends text before `funcdef` (`copyclosure def f(x):...`), modifying the prefix. `yield` inserts inside the body after the first `openindent`. These modify different parts of the string, so order doesn't matter.

## Scope

This fix only addresses **compiler-generated** unreachable code from `yield def`. It does **not** fix user-written patterns like:

```coconut
def it_ret(x):
    return x
    yield None
```

These are correctly flagged as unreachable by the checker. The test functions `it_ret`, `it_ret_none`, and `it_ret_tuple` must still be rewritten to use `if False: yield None` before the `return` to avoid the warning.
