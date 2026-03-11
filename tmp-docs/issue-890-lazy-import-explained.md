# How the Lazy Import Feature Works

This document traces the full lifecycle of a lazy import statement in Coconut — from source syntax, through compilation, down to runtime behaviour — and explains how every involved file contributes.

---

## Overview

A lazy import defers the actual Python `import` call until the name is first used. The two supported forms are:

```coconut
lazy import json                       # module import
lazy from json import dumps            # from-import
```

Five files are involved:

| File | Role |
|------|------|
| `coconut/compiler/grammar.py` | Parses the `lazy` keyword and import syntax |
| `coconut/compiler/compiler.py` | Decides what Python code to emit |
| `coconut/compiler/templates/header.py_template` | The runtime proxy class `_coconut_lazy_module` |
| `coconut/compiler/header.py` | Fills in the template's platform-specific import snippet |
| `coconut/tests/src/cocotest/agnostic/primary_2.coco` | Tests for all lazy import variants |

---

## Step 1 — Parsing (`grammar.py`)

```python
# grammar.py line 2036
import_stmt_ref = Optional(keyword("lazy"), default="") + (from_import | basic_import)
```

The grammar wraps every import statement with an optional `lazy` token. If the keyword is present it is captured as the string `"lazy"`; if absent it defaults to `""`. The rest of the rule is identical to a normal import, so `lazy` works with all forms: plain module import, `from … import`, `as` aliases, and comma-separated names.

The parsed tokens are handed to `import_handle` in `compiler.py`.

---

## Step 2 — Dispatch (`compiler.py` — `import_handle`)

```python
# compiler.py line 4234
def import_handle(self, original, loc, tokens):
    if len(tokens) == 2:
        lazy, imports = tokens          # basic import
    elif len(tokens) == 3:
        lazy, imp_from, imports = tokens  # from-import
    lazy = lazy == "lazy"
    ...
    return self.universal_import(loc, imports, imp_from=imp_from, lazy=lazy)
```

`import_handle` extracts the `lazy` flag and forwards everything to `universal_import`, passing `lazy=True` when the keyword was present. It also enforces restrictions: `lazy import *` and `lazy from __future__` are both syntax errors.

---

## Step 3 — Stdlib compatibility (`compiler.py` — `universal_import`)

```python
# compiler.py line 4166
def universal_import(self, loc, imports, imp_from=None, lazy=False):
```

This method handles Coconut's cross-version stdlib remapping (e.g. `queue` → `Queue` on Python 2). It resolves each import to one or more path strings, then calls `single_import` for each one, threading `lazy` through.

---

## Step 4 — Path normalisation (`compiler.py` — `single_import`)

```python
# compiler.py line 4131
def single_import(self, loc, path, imp_as, type_ignore=False, lazy=False):
    parts = path.split("./")   # "json./dumps" → imp_from="json", imp="dumps"
    ...
    out.append(self.make_import_stmt(imp_from, imp, imp_as, lazy=lazy))
```

`single_import` normalises the internal path format (Coconut uses `"module./attr"` as a separator for from-imports) and handles dotted `as` aliases before delegating to `make_import_stmt`.

---

## Step 5 — Code generation (`compiler.py` — `make_import_stmt`)

This is the decision point. The behaviour depends on three factors: whether `lazy` is set, whether it is a from-import, and whether the target Python version supports PEP 690 native lazy imports (≥ 3.15).

### Python ≥ 3.15

```python
# compiler.py line 4123
else:
    return "lazy from json import dumps"   # emitted verbatim
```

Native `lazy` keyword is passed straight through to the output.

### Python < 3.15 — `lazy import module`

```python
# compiler.py line 4116
'{bind_to} = _coconut_lazy_module("{module}") {type_ignore}'
```

A `ModuleType`-based proxy is created. Before that, `ensure_module_or_create_fake` is called for every intermediate package in a dotted path (e.g. `lazy import os.path` needs a fake `os` module to exist so that `os.path` is accessible as an attribute chain).

### Python < 3.15 — `lazy from module import name` at **function level**

```python
'{bind_to} = _coconut_lazy_module("{module}", attr="{name}") {type_ignore}'
```

A metaclass-based proxy is created and bound to the local name. The proxy stays for the lifetime of the local scope.

### Python < 3.15 — `lazy from module import name` at **module level** *(fixed in this session)*

```python
'{bind_to} = _coconut_lazy_module("{module}", attr="{name}", '
'global_name="{bind_to}", globals_dict=globals()) {type_ignore}'
```

Same proxy, but two extra arguments tell it to **self-replace** in `globals()` on first successful load. After first use, `LOAD_GLOBAL` returns the real object directly, bypassing the proxy on every subsequent access.

The module-level detection uses the parsing context:

```python
scope = self.current_parsing_context("scope")
at_module_level = scope is not None and scope["parent"] is None
```

The root scope is created with `parent=None` inside `inner_environment()`; any function or class scope has a non-`None` parent.

#### Helper — `ensure_module_or_create_fake`

```python
# compiler.py line 4059
def ensure_module_or_create_fake(self, mod_name):
    """Create a fake module if it does not already exist."""
```

Used for dotted module paths only (e.g. `lazy import os.path`). It emits a try/except block that checks whether `os` is already a real `ModuleType`; if not, it creates a `types.ModuleType("os")` placeholder so that `os.path` can be assigned as an attribute. Not used for module-level from-imports since only the leaf name is bound.

---

## Step 6 — The runtime proxy (`header.py_template` + `header.py`)

Every compiled Coconut file begins with a header that includes `_coconut_lazy_module`. The header is generated from `header.py_template`, with one placeholder filled in by `header.py`:

```python
# header.py line 323
lazy_module_import=pycondition(
    (2, 7),
    if_lt='''
import imp
mod = imp.load_module(name, *imp.find_module(name))
    ''',
    if_ge='''
import importlib
mod = importlib.import_module(name)
    ''',
    indent=4,
),
```

This fills the `{lazy_module_import}` slot in the template, choosing between the legacy `imp` API (Python < 2.7) and the modern `importlib` API.

### How `_coconut_lazy_module` works at runtime

```python
def _coconut_lazy_module(name, on_import=None, attr=None, global_name=None, globals_dict=None):
    state = [None, None]   # [module, error]

    def load():
        if state[0] is None and state[1] is None:   # first call only
            try:
                mod = importlib.import_module(name)  # ← filled by header.py
            except ImportError as err:
                state[1] = err                       # cache the error
            else:
                state[0] = mod
                loaded = getattr(mod, attr) if attr else mod
                if global_name is not None and globals_dict is not None:
                    globals_dict[global_name] = loaded   # self-replace in globals()
                # mirror all dunder methods of the real object onto the proxy class
                for cls in type(loaded).__mro__:
                    for dunder, val in vars(cls).items():
                        if dunder.startswith("__") and callable(val):
                            type.__setattr__(lazy_obj, dunder, make_lazy_method(dunder))
                if on_import:
                    on_import(mod)
        if state[1] is not None:
            raise state[1]           # re-raise cached ImportError on every access
        return getattr(state[0], attr) if attr else state[0]
```

**Two proxy bases:**

| Case | `base` | Why |
|------|--------|-----|
| `lazy import module` | `types.ModuleType` | Needs to look like a module (`isinstance(x, ModuleType)` must pass) |
| `lazy from … import name` | `type` (metaclass) | The only way to intercept `__call__`, `__add__`, etc. on a class-like object without knowing the real type in advance |

**Dunder mirroring** — when `load()` first succeeds it copies all dunder methods from the real object's MRO onto the proxy class. This means `repr(lazy_dumps)`, `hash(lazy_dumps)`, and `lazy_dumps(...)` all transparently delegate before the self-replacement has had a chance to fire.

**Self-replacement** (`global_name` + `globals_dict`) — for module-level from-imports, the first successful call to `load()` writes the real object into `globals_dict[global_name]`. From that point on, `LOAD_GLOBAL` in the compiled bytecode finds the real object and never touches the proxy again.

**Deferred `ImportError`** — if the import fails, `state[1]` is set to the error. The proxy remains in `globals()` (self-replacement never happens) and re-raises the `ImportError` on every access, satisfying the "deferred error" contract.

---

## Step 7 — Tests (`primary_2.coco`)

```coconut
# module-level (lines 1–6)
lazy import collections
lazy import \collections.abc
lazy from collections import OrderedDict
lazy from json import encoder as _test_lazy_encoder
lazy from coconut_nonexistent_test_module5 import missing as _test_lazy_missing_ml
```

```coconut
# function-level (inside primary_test_2, lines ~692–759)
lazy import json, types
lazy from json import dumps
lazy from json import loads as ld
lazy import os.path
lazy from os.path import join
lazy import coconut_nonexistent_test_module          # deferred ImportError
lazy from coconut_nonexistent_test_module3 import missing as lazy_missing

# module-level self-replacement assertions
_ = _test_lazy_encoder.ESCAPE                        # trigger load
assert _test_lazy_encoder is json.encoder            # real object now in globals()
assert type(_test_lazy_encoder) is type(json.encoder)
assert_raises(=> _test_lazy_missing_ml.anything, ImportError)
```

The test suite covers:
- Module-level and function-level variants of both forms
- `as` aliases and multiple imports on one line
- Dotted module paths (`os.path`)
- Stdlib remapping (`queue`)
- Deferred `ImportError` for all variants
- Post-load type identity for the module-level self-replacement fix

---

## Data-flow diagram

```
Coconut source
    │
    ▼
grammar.py
  import_stmt_ref
  Optional("lazy") + (from_import | basic_import)
    │  tokens: ["lazy"|"", imp_from?, [imports...]]
    ▼
compiler.py — import_handle
  extracts lazy flag, validates restrictions
    │  lazy=True/False
    ▼
compiler.py — universal_import
  resolves stdlib remaps (queue→Queue etc.)
    │
    ▼
compiler.py — single_import
  normalises "module./attr" path format
    │
    ▼
compiler.py — make_import_stmt
  ┌─ lazy=False ──────────────────────────────────► plain Python import statement
  │
  ├─ lazy=True, target >= 3.15 ───────────────────► "lazy from json import dumps"
  │
  ├─ lazy=True, target < 3.15, lazy import module ► _coconut_lazy_module("json")
  │     + ensure_module_or_create_fake for each
  │       intermediate package in dotted path
  │
  ├─ lazy=True, target < 3.15, from-import,
  │   function level ──────────────────────────────► _coconut_lazy_module("json", attr="dumps")
  │
  └─ lazy=True, target < 3.15, from-import,
      module level ───────────────────────────────► _coconut_lazy_module("json", attr="dumps",
                                                      global_name="dumps",
                                                      globals_dict=globals())
    │
    ▼ (all lazy paths)
header.py_template — _coconut_lazy_module (runtime)
  ┌─ attr=None ────► ModuleType-based proxy
  └─ attr set  ────► metaclass-based proxy
        │
        │ first access
        ▼
      load()
        ├─ importlib.import_module(name)   ← filled by header.py
        ├─ mirror dunders from real object
        ├─ if global_name set: globals_dict[global_name] = loaded   (self-replace)
        └─ cache result in state[0] / error in state[1]
```
