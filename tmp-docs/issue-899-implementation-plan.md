# Implementation Plan: Issue #899 — `**rest` in Data/Class Match Patterns

## Context

Coconut's data/class match patterns (e.g. `match T(a=arg1) in t:`) already support:
- Positional matches: `T(x, y)`
- Named (keyword) matches: `T(a=x, b=y)`
- Starred positional rest: `T(*rest)` → captures remaining positional fields as a tuple

**Missing**: double-starred keyword rest `T(a=x, **rest)` → should capture all unmatched named fields as a dict. Dict matching already supports `{k: v, **rest}`, but the equivalent for class/data patterns throws a parse error. Issue #899 requests adding this feature.

---

## Files to Modify

1. `coconut/compiler/grammar.py` — extend the grammar to parse `**rest` in data match items
2. `coconut/compiler/matching.py` — handle `**rest` tokens in the three match handlers
3. `coconut/tests/src/cocotest/agnostic/suite.coco` — add test cases

---

## Step 1: Grammar (`grammar.py`, line 2072)

**Change** `matchlist_data_item` to add a `dubstar` alternative:

```python
# Before (lines 2072-2075):
matchlist_data_item = (
    matchlist_anon_named_tuple_item
    | Optional(star) + match
)

# After:
matchlist_data_item = (
    matchlist_anon_named_tuple_item
    | dubstar + (setname | condense(lbrace + rbrace))   # **rest or **{}
    | Optional(star) + match
)
```

`dubstar = Literal("**")` is already defined (line 701). The token produced by `dubstar + setname` is `["**", "varname"]`, and for `dubstar + condense(lbrace + rbrace)` it is `["**", "{}"]`. These are length-2 tokens with `[0] == "**"`, unambiguously distinguishable from `star` items (which use `"*"`, disambiguated at parse time via `disambiguate_literal`).

---

## Step 2: Parsing — `split_data_or_class_matches` (`matching.py`, lines 1063–1110)

Add a `dubstar_match` return value and detection branch:

```python
def split_data_or_class_matches(self, matches):
    """Split data/class match tokens into pos_matches, name_matches, star_match, dubstar_match."""
    pos_matches = []
    name_matches = {}
    star_match = None
    dubstar_match = None   # NEW
    for match_arg in matches:
        # positional arg (len == 1) — unchanged
        if len(match_arg) == 1:
            ...

        # NEW: double-starred keyword rest (len == 2, match_arg[0] == "**")
        elif len(match_arg) == 2 and match_arg[0] == "**":
            _, name = match_arg
            if dubstar_match is not None:
                raise CoconutDeferredSyntaxError("duplicate double-starred arg in data/class match", self.loc)
            if star_match is not None:
                raise CoconutDeferredSyntaxError("both starred and double-starred arg in data/class match", self.loc)
            dubstar_match = name

        # starred positional rest (len == 2, match_arg[0] == "*") — add check
        elif len(match_arg) == 2 and match_arg[0] == "*":
            ...
            # add: if dubstar_match is not None -> error

        # keyword arg (else) — add check
        else:
            ...
            if dubstar_match is not None:   # NEW: no keyword args after **rest
                raise CoconutDeferredSyntaxError("keyword arg after double-starred arg in data/class match", self.loc)
            ...

    return pos_matches, name_matches, star_match, dubstar_match   # 4 values now
```

---

## Step 3: Update All Callers of `split_data_or_class_matches`

Three methods call this function and must unpack 4 values:

| Method | Line | Change needed |
|--------|------|---------------|
| `match_data` | 1204 | Unpack `dubstar_match`; add rest-dict logic |
| `match_class` | 1131 | Unpack `dubstar_match`; add rest-dict logic |
| `match_anon_named_tuple` | 1255 | Unpack `dubstar_match`; assert it is `None` |

---

## Step 4: `match_data` handler (`matching.py`, line 1201)

**Semantics of `dubstar_match` values:**

| Value | Meaning |
|-------|---------|
| `None` | No `**rest` → existing strict-length check applies |
| `"{}"` | `**{}` → same strict check, no binding |
| `wildcard` (`"_"`) | Allow extra fields, skip binding |
| `<name>` | Bind remaining named fields as a dict |

**Add after `match_class_names(name_matches, item)` (after line 1223):**

```python
if dubstar_match is not None and dubstar_match != "{}" and dubstar_match != wildcard:
    rest_var = self.get_temp_var()
    excluded = tuple_str_of(name_matches, add_quotes=True)
    self.add_def(
        "{rest_var} = _coconut.dict("
        "(_coconut_k, _coconut.getattr({item}, _coconut_k))"
        " for _coconut_k in _coconut.getattr({item}, '__match_args__', ())[{len_pos}:]"
        " if _coconut_k not in {excluded})".format(
            rest_var=rest_var, item=item,
            len_pos=len(pos_matches), excluded=excluded,
        )
    )
    with self.down_a_level():
        self.match_var([dubstar_match], rest_var)
```

**Modify the strict-length check condition (line 1226):**

```python
# Before:
if star_match is None:

# After:
if star_match is None and (dubstar_match is None or dubstar_match == "{}"):
```

When a named-capture `**rest` is present, the strict-length check is skipped because extra fields are intentionally captured. `**{}` keeps the strict check since the empty dict means no extra fields are allowed.

---

## Step 5: `match_class` handler (`matching.py`, line 1128)

After the `match_class_names` call (around line 1198), add analogous logic.
`match_class` already fetches `__match_args__` from the **class** (not instance) for the `star_match` block (lines 1179–1196); reuse the same approach:

```python
if dubstar_match is not None and dubstar_match != "{}" and dubstar_match != wildcard:
    dubstar_var = self.get_temp_var()
    args_var = self.get_temp_var()
    excluded = tuple_str_of(name_matches, add_quotes=True)
    self.add_def(
        handle_indentation("""
{args_var} = _coconut.getattr({cls_name}, '__match_args__', ())
{dubstar_var} = _coconut.dict(
    (_coconut_k, _coconut.getattr({item}, _coconut_k))
    for _coconut_k in {args_var}[{len_pos}:]
    if _coconut_k not in {excluded}
)
        """).format(
            args_var=args_var, cls_name=cls_name,
            dubstar_var=dubstar_var, item=item,
            len_pos=len(pos_matches), excluded=excluded,
        )
    )
    with self.down_a_level():
        self.match_var([dubstar_match], dubstar_var)
```

`match_data_or_class` delegates to `match_data`/`match_class` directly, so it inherits these changes automatically without modification.

---

## Step 6: Tests (`coconut/tests/src/cocotest/agnostic/suite.coco`)

```coconut
data T(a, b, c)
t = T(1, 2, 3)

# Basic **rest with named match
match T(a=x, **rest) in t:
    assert x == 1
    assert rest == {"b": 2, "c": 3}

# Wildcard **_ (allow extra fields, no binding)
match T(a=x, **_) in t:
    assert x == 1

# **{} strict: all fields must be accounted for
match T(a=1, b=2, c=3, **{}) in t:
    pass  # should match

# **rest with positional + named matches
match T(x, b=y, **rest) in t:
    assert x == 1
    assert y == 2
    assert rest == {"c": 3}

# Non-match: wrong value for named field
def test_dubstar_no_match(t):
    match T(a=99, **rest) in t:
        return False
    else:
        return True
assert test_dubstar_no_match(t)

# class pattern with **rest (PEP-634-style class)
class MyClass:
    __match_args__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y

obj = MyClass(10, 20)
match MyClass(x=xv, **rest) in obj:
    assert xv == 10
    assert rest == {"y": 20}
```

---

## Verification

```bash
# Quick smoke test
python -m coconut -c "
data T(a, b, c)
t = T(1, 2, 3)
match T(a=x, **rest) in t:
    print(x, rest)
"
# Expected: 1 {'b': 2, 'c': 3}

# Full test suite
make test
```
