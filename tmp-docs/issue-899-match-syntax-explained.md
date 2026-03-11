# Coconut Match Syntax Explained

This document covers the match/pattern-matching syntax in Coconut, including its history, how it compares to Python's own match syntax, use cases, and the open issue #899 requesting `**rest` support for data class matching.

---

## History and Origin

Pattern matching was **not invented by Coconut**. It originates from functional programming languages:

- **ML** (1973) — one of the earliest languages with structural pattern matching
- **Haskell** (1990) — pattern matching on constructors is a core feature
- **Erlang** (1986) — pattern matching on terms
- **OCaml** (1996) — `match ... with | pattern -> ...`
- **Scala** (2004) — `match`/`case` syntax
- **F#** (2005) — pattern matching inspired by OCaml

Coconut's docs state it was inspired by Haskell, CoffeeScript, F#, and Julia.

### Timeline in the Python Ecosystem

| Date | Event |
|---|---|
| October 2014 | Coconut initial commit |
| March 2015 | Coconut adds `match` statements |
| October 2021 | Python 3.10 adds structural pattern matching (PEP 634) |

Coconut introduced pattern matching to the Python ecosystem **~6.5 years before Python itself** adopted it. Python's PEP 634 was independently inspired by the same ML/Haskell tradition, not necessarily by Coconut.

---

## Coconut vs Python Match Syntax

Coconut's match syntax is a **strict superset** of Python's (PEP 634). Coconut's `case` block style is deliberately compatible with Python 3.10 syntax and compiles to work on all Python versions.

### Syntax Difference

**Python (PEP 634):**
```python
match value:
    case 0:
        return "zero"
    case int(n) if n > 0:
        return "positive"
```

**Coconut — standalone style:**
```coconut
match 0 in value:           return "zero"
match int() as n in value if n > 0:  return "positive"
```

**Coconut — case-block style (compatible with Python 3.10):**
```coconut
match value:
    case 0:           return "zero"
    case int(n) if n > 0:  return "positive"
```

### Shared Features

| Feature | Python | Coconut |
|---|---|---|
| Constant matching (`0`, `True`, `"x"`) | ✅ | ✅ |
| Type/class matching (`int()`, `MyClass(x)`) | ✅ | ✅ |
| Sequence destructuring (`[a, b]`, `(a, b)`) | ✅ | ✅ |
| Dict matching (`{"k": v}`) | ✅ | ✅ |
| OR patterns (`A \| B`) | ✅ | ✅ |
| Guard (`if cond`) | ✅ | ✅ |
| Wildcard (`_`) | ✅ | ✅ |
| `as` binding | ✅ | ✅ |

### What Coconut Adds on Top

| Feature | Example |
|---|---|
| `match ... in ...` standalone syntax | `match 0 in x:` |
| Iterator/lazy matching | `match (| |) in x:` (works on any `Iterable`) |
| Head-tail / search splits | `match [1] + m + [5] in l:` |
| String splits | `match s1 + "," + s2 in s:` |
| View patterns | `match plus1 -> 6 in x:` |
| Infix checks | `` match n `isinstance` int in x: `` |
| `data` type matching (algebraic types) | `match tree(l, r) in t:` |
| Destructuring assignment | `[head] + tail = mylist` |
| Pattern-matching functions | `match def f([head] + tail) = head` |
| `addpattern` multi-case functions | `addpattern def f(0) = 1` |
| Works on **all** Python versions (not just 3.10+) | — |

### Notable Incompatibility

Python uses dotted names (e.g. `Foo.BAR`) as implicit equality checks in patterns. Coconut requires `==Foo.BAR` for that — this is flagged as a style warning in `--strict` mode.

---

## Match Syntax Use Cases

### 1. Basic Constant and Type Matching

```coconut
def factorial(value):
    match 0 in value:                         # exact constant
        return 1
    match int() as n in value if n > 0:       # type check + guard
        return n * factorial(n - 1)
```

### 2. Tuple / List Destructuring

```coconut
def classify(value):
    match () in value:              return "empty tuple"
    match (_,) in value:            return "singleton"
    match (x, x) in value:          return "duplicate pair of " + str(x)
    match (_,_) in value:           return "pair"
    match [x] + xs in value:        return "list head=" + str(x)
    match _ + [a, b] in value:      return "last two"
```

`x, x` requires both elements to be equal. `_` is a wildcard.

### 3. Head-Tail and Init-Last Splits

```coconut
def head_tail(l):
    match [head] + tail = l    # destructuring assignment
    return head, tail

def init_last(l):
    init + [last] = l
    return init, last

def one_to_five(l):
    match [1] + m + [5] in l:  # head-last split: captures middle
        return m
    else:
        return False
```

### 4. Iterable / Lazy List Matching

```coconut
def list_type(xs):
    cases reiterable(xs):           # safe for iterators
        match [fst, snd] :: tail:   return "at least 2"
        match [fst] :: tail:        return "at least 1"
        match (| |):                return "empty"
```

`::` is iterable split (works on any `Iterable`, unlike `+` which requires a `Sequence`). `(| |)` is an empty lazy list.

### 5. Dict Matching with `**rest`

```coconut
match def method(cls, {'somekey': str()}) = True

# with rest capture:
match def method(cls, {'someotherkey': int(), **rest}) =
    super().method(rest)
```

`**rest` captures all unmatched keys into a dict.

### 6. Set Matching

```coconut
def classify(value):
    match s{*_} in value:           # any set
        match s{*()} in value:      # fixed-length: empty set
            return "empty set"
        match {0, *()} in value:    # fixed-length: exactly {0}
            return "set of 0"
        return "set"
```

`s{*_}` matches any set; `s{*()}` enforces no extra elements.

### 7. Data Type Matching (Positional, Named, Starred)

```coconut
data empty()
data leaf(n)
data node(l, r)
tree = (empty, leaf, node)

def depth(t):
    match tree() in t:          return 0        # any type in tree tuple
    match tree(n) in t:         return 1        # positional arg
    match tree(l, r) in t:      return 1 + max(depth(l), depth(r))

# named args:
def depth2(t):
    match tree(n=n) in t:       return 1
    match tree(l=l, r=r) in t:  return 1 + max(depth2(l), depth2(r))

# starred rest (positional):
data Node(*children)
def tree_depth(Leaf(_)) = 0
addpattern def tree_depth(Node(*children)) =
    children |> map$(tree_depth) |> max |> (.+1)
```

### 8. View Patterns (Transform then Match)

```coconut
plus1 = (+ 1)

plus1 -> 4 = 3              # apply plus1 to 3, check result == 4
plus1 -> x = 5              # apply plus1 to 5, bind result to x (x == 6)
(plus1..plus1) -> 5 = 3    # compose functions then match

match plus1 -> 6 in 3:     # won't match: plus1(3)==4, not 6
    assert False
```

### 9. String Search Splits

```coconut
def has_abc(s):
    match _ + "abc" + _ in s:  # find "abc" anywhere in string
        return True
    else:
        return False

def split1_comma(s):
    match s1 + "," + s2 in s:  # split on first comma
        return s1, s2
    else:
        return s, ""

# twin primes via iterable search split:
def first_twin(_ + [p, (.-2) -> p] + _) = (p, p+2)
# finds two consecutive elements differing by 2
```

### 10. `case def` / `addpattern` for Multi-Case Functions

```coconut
case def depth:
    case(tree()) = 0
    case(tree(n=n)) = 1
    case(tree(l=l, r=r)) = 1 + max(depth(l), depth(r))

# addpattern: define each case separately
match def fact(n) = fact(n, 1)
match addpattern def fact(0, acc) = acc
addpattern match def fact(n, acc) = fact(n-1, acc*n)
```

### 11. Infix Patterns / Isinstance Checks

```coconut
match int() as n in value if n > 0: ...
# equivalent using infix:
match n `isinstance` int in value: ...

# strict attribute matching with dot prefix:
match clsA(.a=2) in obj:   # raises AttributeError if .a doesn't exist
```

### 12. OR Patterns

```coconut
match (_,_,_) or (_,_,_,_) in value:   # matches 3- or 4-element tuple
    return "few"

AccessCounter(x=1) or AccessCounter(x=1) = ac  # tries first, then second
```

---

## Issue #899: Missing `**rest` in Data Class Matching

### What the Issue Requests

```coconut
data T(a: int, b: int)
t = T(1, 2)

match T(a=arg1, **_) in t: print(arg1)   # currently throws a parse error
```

### The Three Match Patterns Involved

**1. Data type matching with named args** — already works:
```coconut
match T(a=arg1) in t:
    print(arg1)
```

**2. Dict matching with `**rest`** — already works:
```coconut
match {"a": 1, **rest} in d:   # captures remaining keys into `rest`
    print(rest)
```

**3. Data type matching with `**rest`** — NOT yet supported (the feature request):
```coconut
match T(a=arg1, **rest) in t:
    print(arg1)   # 1
    print(rest)   # {"b": 2}  ← desired behavior
```

### Why the Parse Error Occurs

The data type match grammar (`grammar.py:2072–2076`) only allows:

```python
matchlist_data_item = (
    matchlist_anon_named_tuple_item   # name=pattern
    | Optional(star) + match          # *rest or plain pattern
)
```

It supports `*rest` (starred positional rest) but **not `**rest`** (double-starred keyword rest). The `dubstar` (`**`) token is absent from `matchlist_data_item`.

By contrast, dict matching at `grammar.py:2152` explicitly includes:
```python
Optional(dubstar + (setname | condense(lbrace + rbrace)))
```

### The Fix Would Require

1. Adding `dubstar` to `matchlist_data_item` in `grammar.py`
2. Updating the match handler in `matching.py` to collect unmatched named attributes into a dict when `**rest` is present
