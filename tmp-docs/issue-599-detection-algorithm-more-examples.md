# `detect_unreachable_code` — More Algorithm Walkthroughs

This document applies the same step-by-step trace from `issue-599-detection-algorithm-explain.md`
to four additional cases drawn from `issue-599-more-test-case-impl-plan.md`.

---

## Reminder: Gate Condition

The analysis gate fires only when **all three** hold:

```
level == 1  AND  disabled_until_level is None  AND  base is non-empty
```

`func_until_level` is NOT checked directly by the gate — it is only used to determine
whether a subsequent `def` re-arms the outer `disabled_until_level`.
When a nested `def` is seen, **both** `func_until_level = level` and
`disabled_until_level = level` are set (if `disabled_until_level` was None).
This means the gate is always suppressed while inside a nested function.

---

## Case 1 — `raise` terminator (Positive: error expected)

### Code

```python
def f():
    raise ValueError
    x = 2
```

### `raw_lines` (simplified)

| # | Internal line | Indent | Body | Dedent |
|---|---------------|--------|------|--------|
| 1 | `<INDENT>raise ValueError # 2` | +1 | `raise ValueError` | — |
| 2 | `x = 2<DEDENT> # 3` | — | `x = 2` | −1 |

### Initial State

```
level = 0 | func_until_level = None | disabled_until_level = None | last_terminator = None
```

---

### Line 1 — `raise ValueError`

**Step 1 — leading indent (+1)**
```
level = 0 + 1 = 1
```

**Step 2 — scope-exit checks** (both trackers None, nothing to do)

**Step 3 — scope-entry checks**
- `def_regex.match("raise ValueError")` → no match
- `tco_disable_regex.match("raise ValueError")` → no match

**Step 4 — gate** (level == 1, disabled_until_level is None, base non-empty → **enter**)
- `last_terminator is None` → no error
- `terminator_stmt_regex.match("raise ValueError")` → **MATCH**, `group(1) = "raise"`
  - `last_terminator = ("raise", 2)` ← terminator recorded

**Step 5 — trailing dedent** (none)

**State after line 1:**
```
level = 1 | func_until_level = None | disabled_until_level = None | last_terminator = ("raise", 2)
```

---

### Line 2 — `x = 2`

**Step 1 — leading indent** (none)
```
level = 1  (unchanged)
```

**Step 2 — scope-exit checks** (both None, nothing to do)

**Step 3 — scope-entry checks** (no matches)

**Step 4 — gate** (level == 1, disabled_until_level is None, base non-empty → **enter**)
- `last_terminator = ("raise", 2)` → **NOT None** → 🚨 **ERROR REPORTED**:
  ```
  CoconutStyleError: found unreachable code after raise statement (line 2)
  ```
  - `last_terminator = None` (reset)
- `terminator_stmt_regex.match("x = 2")` → no match → `last_terminator = None`

**Step 5 — trailing dedent (−1)**
```
level = 1 - 1 = 0
```

**State after line 2:**
```
level = 0 | func_until_level = None | disabled_until_level = None | last_terminator = None
```

### Summary

| Line | `level` | `disabled_until_level` | `last_terminator` entering gate | Gate action |
|------|---------|------------------------|---------------------------------|-------------|
| `raise ValueError` | 1 | None | None | no error; record `("raise", 2)` |
| `x = 2` | 1 | None | `("raise", 2)` | **🚨 error**; reset |

**Why it differs from the `return` example**: The keyword captured in `group(1)` is `"raise"` instead of `"return"`, producing the message `"after raise statement"`. The detection logic is otherwise identical.

---

## Case 2 — Nested `def` (Negative: no error expected)

### Code

```python
def f():
    def g():
        return 1
    x = 2
```

### `raw_lines` (simplified)

| # | Internal line | Indent | Body | Dedent |
|---|---------------|--------|------|--------|
| 1 | `<INDENT>def g(): # 2` | +1 | `def g():` | — |
| 2 | `<INDENT>return 1<DEDENT> # 3` | +1 | `return 1` | −1 |
| 3 | `x = 2<DEDENT> # 4` | — | `x = 2` | −1 |

### Initial State

```
level = 0 | func_until_level = None | disabled_until_level = None | last_terminator = None
```

---

### Line 1 — `def g():`

**Step 1 — leading indent (+1)**
```
level = 0 + 1 = 1
```

**Step 2 — scope-exit checks** (both None, nothing to do)

**Step 3 — scope-entry checks**
- `def_regex.match("def g():")` → **MATCH**
  - `func_until_level = 1`  ← set to current level
  - `disabled_until_level = 1` ← also set (was None) ← **this is why the gate is suppressed**

**Step 4 — gate**: `disabled_until_level = 1` (NOT None) → **GATE SKIPPED**

**Step 5 — trailing dedent** (none)

**State after line 1:**
```
level = 1 | func_until_level = 1 | disabled_until_level = 1 | last_terminator = None
```

---

### Line 2 — `return 1` (inside `g`, level 2)

**Step 1 — leading indent (+1)**
```
level = 1 + 1 = 2
```

**Step 2 — scope-exit checks**
- `func_until_level = 1`, `level = 2` → `2 <= 1`? **No** → stays 1
- `disabled_until_level = 1`, `level = 2` → `2 <= 1`? **No** → stays 1

**Step 3 — scope-entry checks**
- `def_regex.match("return 1")` → no match
- `disabled_until_level` is not None → tco_disable_regex check skipped

**Step 4 — gate**: `level == 2` ≠ 1 → **GATE SKIPPED**

**Step 5 — trailing dedent (−1)**
```
level = 2 - 1 = 1
```

**State after line 2:**
```
level = 1 | func_until_level = 1 | disabled_until_level = 1 | last_terminator = None
```

---

### Line 3 — `x = 2`

**Step 1 — leading indent** (none)
```
level = 1  (unchanged)
```

**Step 2 — scope-exit checks**
- `func_until_level = 1`, `level = 1` → `1 <= 1`? **YES** → `func_until_level = None`
- `disabled_until_level = 1`, `level = 1` → `1 <= 1`? **YES** → `disabled_until_level = None`

Both trackers cleared — we have returned to outer function scope.

**Step 3 — scope-entry checks** (no matches for `x = 2`)

**Step 4 — gate** (level == 1, disabled_until_level is None → **enter**)
- `last_terminator is None` → **no error** ✓
- `terminator_stmt_regex.match("x = 2")` → no match → `last_terminator = None`

**Step 5 — trailing dedent (−1)**
```
level = 1 - 1 = 0
```

### Summary

| Line | `level` | `disabled_until_level` | Gate entered? | Action |
|------|---------|------------------------|---------------|--------|
| `def g():` | 1 | **1** (just set) | No | Suppress both trackers |
| `return 1` | 2 | 1 | No (level ≠ 1) | — |
| `x = 2` | 1 | **None** (cleared) | Yes | no error; last_terminator was None |

**Key insight**: Setting `disabled_until_level = level` on entry to the nested `def` line itself means the gate is already suppressed on that very line. The `return` inside `g` (at level 2) is additionally excluded by the `level == 1` guard. When level returns to 1 on `x = 2`, both trackers are cleared first, so `x = 2` is analysed cleanly with `last_terminator = None`.

---

## Case 3 — `for` Loop (Negative: no error expected)

### Code

```python
def f():
    for i in range(10):
        return i
    x = 2
```

### `raw_lines` (simplified)

| # | Internal line | Indent | Body | Dedent |
|---|---------------|--------|------|--------|
| 1 | `<INDENT>for i in range(10): # 2` | +1 | `for i in range(10):` | — |
| 2 | `<INDENT>return i<DEDENT> # 3` | +1 | `return i` | −1 |
| 3 | `x = 2<DEDENT> # 4` | — | `x = 2` | −1 |

### Initial State

```
level = 0 | func_until_level = None | disabled_until_level = None | last_terminator = None
```

---

### Line 1 — `for i in range(10):`

**Step 1 — leading indent (+1)**
```
level = 0 + 1 = 1
```

**Step 2 — scope-exit checks** (both None, nothing to do)

**Step 3 — scope-entry checks**
- `def_regex.match("for i in range(10):")` → no match
- `disabled_until_level is None` AND `tco_disable_regex.match("for i in range(10):")` → **MATCH**
  - `disabled_until_level = 1`

**Step 4 — gate**: `disabled_until_level = 1` (NOT None) → **GATE SKIPPED**

**Step 5 — trailing dedent** (none)

**State after line 1:**
```
level = 1 | func_until_level = None | disabled_until_level = 1 | last_terminator = None
```

---

### Line 2 — `return i` (inside loop, level 2)

**Step 1 — leading indent (+1)**
```
level = 1 + 1 = 2
```

**Step 2 — scope-exit checks**
- `disabled_until_level = 1`, `level = 2` → `2 <= 1`? No → stays 1

**Step 3 — scope-entry checks**
- `def_regex` → no match
- `disabled_until_level` not None → tco_disable_regex check skipped

**Step 4 — gate**: `level == 2` ≠ 1 → **GATE SKIPPED**

**Step 5 — trailing dedent (−1)**
```
level = 2 - 1 = 1
```

**State after line 2:**
```
level = 1 | func_until_level = None | disabled_until_level = 1 | last_terminator = None
```

---

### Line 3 — `x = 2`

**Step 1 — leading indent** (none)
```
level = 1  (unchanged)
```

**Step 2 — scope-exit checks**
- `disabled_until_level = 1`, `level = 1` → `1 <= 1`? **YES** → `disabled_until_level = None`

**Step 3 — scope-entry checks** (no matches)

**Step 4 — gate** (level == 1, disabled_until_level is None → **enter**)
- `last_terminator is None` → **no error** ✓
- `terminator_stmt_regex.match("x = 2")` → no match → `last_terminator = None`

**Step 5 — trailing dedent (−1)**
```
level = 1 - 1 = 0
```

### Summary

| Line | `level` | `disabled_until_level` | Gate entered? | Action |
|------|---------|------------------------|---------------|--------|
| `for i in range(10):` | 1 | **1** (just set) | No | — |
| `return i` | 2 | 1 | No (level ≠ 1) | — |
| `x = 2` | 1 | **None** (cleared) | Yes | no error |

**Key insight**: `for` is handled by `tco_disable_regex` (not `def_regex`), so only `disabled_until_level` is set (not `func_until_level`). The outcome is the same: the gate is suppressed for the loop body. A `return` inside a loop is valid — the loop may not execute every iteration, so code after the loop is always reachable.

**Contrast with nested `def`**: For `for`, only `disabled_until_level` is set. For `def`, both `func_until_level` and `disabled_until_level` are set. The gate only checks `disabled_until_level`, so the effect on the gate is identical.

---

## Case 4 — `if`/`else` with All Branches Returning (Negative: no error expected)

### Code

```python
def f():
    if True:
        return 1
    else:
        return 2
    x = 3
```

### `raw_lines` (simplified)

| # | Internal line | Indent | Body | Dedent |
|---|---------------|--------|------|--------|
| 1 | `<INDENT>if True: # 2` | +1 | `if True:` | — |
| 2 | `<INDENT>return 1<DEDENT> # 3` | +1 | `return 1` | −1 |
| 3 | `else: # 4` | — | `else:` | — |
| 4 | `<INDENT>return 2<DEDENT> # 5` | +1 | `return 2` | −1 |
| 5 | `x = 3<DEDENT> # 6` | — | `x = 3` | −1 |

### Initial State

```
level = 0 | func_until_level = None | disabled_until_level = None | last_terminator = None
```

---

### Line 1 — `if True:`

**Step 1 — leading indent (+1)**
```
level = 0 + 1 = 1
```

**Step 2 — scope-exit checks** (both None, nothing to do)

**Step 3 — scope-entry checks**
- `def_regex.match("if True:")` → no match
- `tco_disable_regex.match("if True:")` → **no match** ← `if` is deliberately excluded

**Step 4 — gate** (level == 1, disabled_until_level is None → **enter**)
- `last_terminator is None` → no error
- `terminator_stmt_regex.match("if True:")` → no match → `last_terminator = None`

**Step 5 — trailing dedent** (none)

**State after line 1:**
```
level = 1 | disabled_until_level = None | last_terminator = None
```

---

### Line 2 — `return 1` (inside `if`, level 2)

**Step 1 — leading indent (+1)**
```
level = 1 + 1 = 2
```

**Step 2 — scope-exit checks** (both None)

**Step 3 — scope-entry checks** (no matches)

**Step 4 — gate**: `level == 2` ≠ 1 → **GATE SKIPPED** ← `return` at level 2 is invisible to the detector

**Step 5 — trailing dedent (−1)**
```
level = 2 - 1 = 1
```

**State after line 2:**
```
level = 1 | disabled_until_level = None | last_terminator = None
```

---

### Line 3 — `else:`

**Step 1 — leading indent** (none)
```
level = 1  (unchanged)
```

**Step 2 — scope-exit checks** (both None)

**Step 3 — scope-entry checks**
- `def_regex.match("else:")` → no match
- `tco_disable_regex.match("else:")` → no match

**Step 4 — gate** (level == 1, disabled_until_level is None → **enter**)
- `last_terminator is None` → no error
- `terminator_stmt_regex.match("else:")` → no match → `last_terminator = None`

**Step 5 — trailing dedent** (none)

**State after line 3:**
```
level = 1 | disabled_until_level = None | last_terminator = None
```

---

### Line 4 — `return 2` (inside `else`, level 2)

**Step 1 — leading indent (+1)**
```
level = 1 + 1 = 2
```

**Step 4 — gate**: `level == 2` ≠ 1 → **GATE SKIPPED** ← again invisible

**Step 5 — trailing dedent (−1)**
```
level = 2 - 1 = 1
```

**State after line 4:**
```
level = 1 | disabled_until_level = None | last_terminator = None
```

---

### Line 5 — `x = 3`

**Step 1 — leading indent** (none)
```
level = 1  (unchanged)
```

**Step 2 — scope-exit checks** (both None)

**Step 3 — scope-entry checks** (no matches)

**Step 4 — gate** (level == 1, disabled_until_level is None → **enter**)
- `last_terminator is None` → **no error** ✓ ← conservative design
- `terminator_stmt_regex.match("x = 3")` → no match → `last_terminator = None`

**Step 5 — trailing dedent (−1)**
```
level = 1 - 1 = 0
```

### Summary

| Line | `level` | `disabled_until_level` | `last_terminator` entering gate | Gate action |
|------|---------|------------------------|---------------------------------|-------------|
| `if True:` | 1 | None | None | no error; no match; stays None |
| `return 1` | 2 | None | — | SKIPPED (level ≠ 1) |
| `else:` | 1 | None | None | no error; no match; stays None |
| `return 2` | 2 | None | — | SKIPPED (level ≠ 1) |
| `x = 3` | 1 | None | **None** | **no error** |

**Key insight — conservative design**: `if`/`else` are intentionally absent from both `def_regex` and `tco_disable_regex`. The `return` statements inside the branches are at level 2, so they never set `last_terminator`. When `x = 3` is reached at level 1, `last_terminator` is still None and no error fires — even though both branches return. The detector does **no branch analysis**; detecting this case would require proving that all branches terminate, which is a significantly harder problem.

---

## Side-by-Side Comparison

| Test case | Suppression mechanism | Why no false positive |
|---|---|---|
| `raise` + code (Case 1) | — | Error fires correctly |
| nested `def g()` (Case 2) | `disabled_until_level = 1` set by `def_regex` | `return` inside `g` is at level 2 AND gate is suppressed at level 1 |
| `for` loop (Case 3) | `disabled_until_level = 1` set by `tco_disable_regex` | `return` inside loop is at level 2 AND gate suppressed at level 1 |
| `if`/`else` all return (Case 4) | No suppression needed | `return` in branches is at level 2, never seen by gate; no branch analysis |
