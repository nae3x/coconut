# Issue #899 Implementation Log — `**rest` in Data/Class Match Patterns

## Summary

Added support for double-starred keyword rest (`**rest`) in Coconut data/class match patterns, enabling syntax like `match T(a=x, **rest) in t:`.

---

## Files Modified

1. `coconut/compiler/grammar.py` — grammar rule for `matchlist_data_item`
2. `coconut/compiler/matching.py` — `split_data_or_class_matches`, `match_class`, `match_data`, `match_anon_named_tuple`
3. `coconut/tests/src/cocotest/agnostic/suite.coco` — new test cases

---

## Changes Made

### 1. `grammar.py` (line 2072)

Extended `matchlist_data_item` to accept a `dubstar` alternative before the existing starred/match alternatives:

```python
matchlist_data_item = (
    matchlist_anon_named_tuple_item
    | dubstar + (setname | condense(lbrace + rbrace))   # NEW: **rest or **{}
    | Optional(star) + match
)
```

`dubstar = Literal("**")` was already defined. The resulting tokens `["**", name]` are length-2 with `[0] == "**"`, unambiguous from `["*", match]` starred items.

### 2. `matching.py` — `split_data_or_class_matches`

- Added `dubstar_match = None` as a fourth return value
- Added detection branch for `len(match_arg) == 2 and match_arg[0] == "**"` before the `"*"` branch
- Added validation: duplicate `**`, both `*` and `**`, and keyword arg after `**` are all errors
- Added check in the `*` branch: error if `dubstar_match` is already set

### 3. `matching.py` — `match_class`

- Updated unpacking to 4 values: `pos_matches, name_matches, star_match, dubstar_match`
- Added block after `star_match` handling: when `dubstar_match` is not None, not `"{}"`, and not `wildcard`, emit code that:
  1. Gets `__match_args__` from the **class**
  2. Builds a dict of `{k: getattr(item, k)}` for all `__match_args__[num_pos_matches:]` keys not in `name_matches`
  3. Calls `match_var` to bind/match the dict

### 4. `matching.py` — `match_data`

- Updated unpacking to 4 values
- Added block after `star_match` handling: when `dubstar_match` is not None, not `"{}"`, and not `wildcard`, emit code that:
  1. Builds a dict of `{k: getattr(item, k)}` for all `item.__match_args__[num_pos_matches:]` keys not in `name_matches`
  2. Calls `match_var` to bind/match the dict
- Modified the strict length check guard:
  - Before: `if star_match is None:`
  - After: `if star_match is None and (dubstar_match is None or dubstar_match == "{}"):`
  - This allows `**{}` to still trigger the strict check (no unmatched fields allowed), while a named `**rest` or `**_` disables it

### 5. `matching.py` — `match_anon_named_tuple`

- Updated unpacking to 4 values
- Updated `internal_assert` to also assert `dubstar_match is None` (not valid in anonymous named tuple patterns)

---

## Problems Found During Implementation vs Plan

### Problem 1: `wildcard` check condition in `match_data` guard

**Plan said:** Use `if star_match is None and dubstar_match not in (wildcard, <name>):` or equivalently allow strict check for `dubstar_match == "{}"`.

**Reality:** The condition `dubstar_match not in (wildcard, <name>)` is tricky because `<name>` covers any variable name, not just a fixed value. The correct approach is the complement: skip the strict check when we have a named-capture `**rest` OR `**_`, but still enforce it when `**{}` is used (since `**{}` explicitly demands no leftover fields).

Final guard used:
```python
if star_match is None and (dubstar_match is None or dubstar_match == "{}"):
```

This is semantically cleaner than the plan's formulation.

### Problem 2: `match_class` dubstar scope

**Plan** mentioned the `star_match` block in `match_class` fetches `__match_args__` inside `self.add_def`. The `dubstar_match` block similarly needs to get `__match_args__` separately, since it may run independently of `star_match`.

The implementation uses two separate `get_temp_var()` calls for `dubstar_args_var` (to hold `__match_args__`) and `dubstar_var` (to hold the resulting dict), matching the existing pattern for `star_match`.

### Problem 3: `**{}` semantics

**Plan** described `**{}` as "same as None (enforces no unmatched fields; no dict binding)". The implementation achieves this by keeping `**{}` excluded from the dubstar binding block but NOT modifying the strict-check guard for it — so `**{}` still falls into the `if star_match is None and (dubstar_match is None or dubstar_match == "{}"):` branch and the existing data-defaults strict check runs. This correctly rejects matches with leftover fields.

### Problem 4: Missing dev dependencies

The test environment was missing several packages (`cPyparsing`, `prompt_toolkit`, `pygments`, `setuptools`, `pytest`, `typing_extensions`, `async_generator`, `anyio`). These needed to be installed before `make test` could run. This is a setup issue unrelated to the implementation.

---

## Test Cases Added (`suite.coco`)

```coconut
# **rest captures unmatched named fields
match data vector(x=xv, **drest) in v:
    assert xv == 1
    assert drest == {"y": 2}

# wildcard **_ ignores rest
match data vector(x=xv, **_) in v:
    assert xv == 1

# **{} strict: all fields must be matched
match data vector(x=1, y=2, **{}) in v:
    pass  # matches

# **{} fails if unmatched fields remain
match data vector(x=1, **{}) in v:
    assert False  # should not match

# non-match: wrong value
match data vector(x=99, **drest) in v:
    assert False  # should not match

# class pattern **rest
match class Matchable(x=xv, **drest) in m:
    assert xv == 1
    assert drest == {"y": 2, "z": 3}

# class pattern **_ wildcard
match class Matchable(x=xv, **_) in m:
    assert xv == 1

# mixed positional + keyword + **rest in class pattern
match class Matchable(1, z=zv, **drest) in m:
    assert zv == 3
    assert drest == {"y": 2}
```

All tests pass with `<success>` at the end of `runner.py`.
