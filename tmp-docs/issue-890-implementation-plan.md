# Plan: Fix `lazy from module import name` (Issue #890)

## Context

Coconut's `lazy import module` works correctly — it creates a `ModuleType`-based proxy. But `lazy from module import name` is "still buggy": it creates a **metaclass-based proxy** (`base = type`) that permanently occupies the name in `globals()`. The proxy never gets replaced with the real imported object, so:

- `type(dumps)` always returns the proxy metaclass, not `function`
- Tools like `inspect.isfunction`, pickling, etc., see the wrong type
- The name in `globals()` is permanently a fake class, not the real object

The issue owner says: **"detect when the name is referenced and then import it there rather than mocking it"** — meaning: when the name is first used, actually do the real import and replace the binding with the real object, not just mock it with a permanent proxy.

## Root Cause Analysis

The current `_coconut_lazy_module(name, attr="attr")` proxy (for from-imports):
1. Creates a class inheriting from `type` (metaclass) bound to `dumps` in globals
2. When `dumps(...)` is called, the proxy's `__call__` runs `load()` to get `json.dumps` and calls it
3. BUT `globals()["dumps"]` remains the proxy class forever — the real `json.dumps` is never cached there
4. All future `LOAD_GLOBAL "dumps"` still go through the proxy

The fix: make `load()` **self-replace** `globals()[name]` with the real object on first successful load. After that, `LOAD_GLOBAL "dumps"` gets `json.dumps` directly.

## Key Files

- `coconut/compiler/templates/header.py_template` — `_coconut_lazy_module()`, add `global_name` + `globals_dict` params
- `coconut/compiler/compiler.py` — `make_import_stmt()`, detect module level and pass extra params
- `coconut/tests/src/cocotest/agnostic/primary_2.coco` — lazy import tests, add module-level assertions

## Implementation

### Step 1: Extend `_coconut_lazy_module` with self-replacement

In `header.py_template`, add `global_name=None, globals_dict=None` parameters. In the `else` block of `load()` (after successful import), add the self-replacement before the dunder setup:

```python
def _coconut_lazy_module(name, on_import=None, attr=None, global_name=None, globals_dict=None):
    state = [None, None]
    def load():
        if state[0] is None and state[1] is None:
            try:
{lazy_module_import}
            except _coconut.ImportError as err:
                state[1] = err
            else:
                state[0] = mod
                loaded = _coconut.getattr(state[0], attr) if attr is not None else state[0]
                # Self-replace in globals on first successful load (for module-level lazy from-imports)
                if global_name is not None and globals_dict is not None:
                    globals_dict[global_name] = loaded
                for cls in _coconut_type(loaded).__mro__:
                    ...  # (unchanged dunder setup)
```

**Effect:** On first use of `dumps`, `globals()["dumps"]` becomes `json.dumps` (the real function). All future `LOAD_GLOBAL "dumps"` bypass the proxy entirely.

**On ImportError:** `globals_dict[global_name]` is NOT set (stays as proxy), so `LOAD_GLOBAL "dumps"` still returns the proxy, which still raises `ImportError` on each access. ✓

### Step 2: Detect module level in `make_import_stmt`

In `make_import_stmt` in `compiler.py`, detect whether we're at module level using the parsing context:

```python
scope = self.current_parsing_context("scope")
at_module_level = scope is not None and scope["parent"] is None
```

At module level, the scope's `parent` is `None` (set in `inner_environment` via `get_empty_scope(inner=True)`). Inside a function/class, `scope["parent"]` points to the enclosing scope (not None).

### Step 3: Pass `global_name` and `globals_dict` for module-level from-imports

In `make_import_stmt`, inside the `if lazy and self.target_info < (3, 15):` block, split on whether it's a from-import at module level:

```python
if imp_from is not None and at_module_level:
    # Module-level from-import: self-replacing proxy
    out_lines.append(
        '{bind_to} = _coconut_lazy_module("{module}", attr="{attr}", '
        'global_name="{bind_to}", globals_dict=globals()) {type_ignore}'.format(...)
    )
else:
    # Function-level from-import OR module import: keep existing proxy behavior
    ...
```

Note: for module-level from-imports, we **skip** `ensure_module_or_create_fake` — it creates a fake parent module which is unnecessary when we're only binding the attr directly.

### Step 4: Update tests in `primary_2.coco`

Add at module level (top of file):
```coconut
lazy from json import encoder as _test_lazy_encoder
lazy from coconut_nonexistent_test_module5 import missing as _test_lazy_missing_ml
```

Add assertions in the test function verifying module-level behavior:
```coconut
# module-level lazy from-import: after first use, globals()[name] is the real object
_ = _test_lazy_encoder.ESCAPE  # trigger load
assert _test_lazy_encoder is json.encoder
assert type(_test_lazy_encoder) is type(json.encoder)

# module-level deferred ImportError
assert_raises(=> _test_lazy_missing_ml.anything, ImportError)
```

## What Changes and What Doesn't

| Case | Before | After |
|------|--------|-------|
| `lazy import module` (module-level) | Proxy forever | Unchanged (already works) |
| `lazy from module import name` (module-level) | Proxy forever | Self-replacing proxy: real object in globals after first use |
| `lazy from module import name` (function-level) | Proxy forever | Unchanged (proxy, acceptable for local vars) |
| `lazy from module import name` (Python 3.15+) | Native `lazy` keyword | Unchanged |
| Deferred ImportError (all) | Proxy raises | Proxy raises (globals not replaced on failure) ✓ |

## Verification

```bash
# Quick test: verify module-level self-replacement
python -m coconut -c "
lazy from json import dumps
print(type(dumps))        # Should print proxy class BEFORE first use
x = dumps([1, 2, 3])
print(type(dumps))        # Should print <class 'builtin_function_or_method'> AFTER first use
print(dumps is __import__('json').dumps)  # Should print True
"

# Full test suite
make test
```
