# `detect_unreachable_code` — Algorithm Walkthrough

This document traces through the `detect_unreachable_code` method step by step,
using the following concrete example:

```python
def func():
    do_stuff()
    return 1
    do_more_stuff()  # ← should be flagged as unreachable
```

---

## How `raw_lines` Looks

The method is called from `proc_funcdef` **after** the `def func():` line has already
been removed. What remains in `raw_lines` is the function body, with Coconut's
internal indent/dedent markers embedded in each line.

Conceptually, the three body lines look like:

| # | Internal line (simplified) | Indent marker | Body | Dedent marker |
|---|---------------------------|---------------|------|---------------|
| 1 | `<INDENT>do_stuff()` | `<INDENT>` (+1) | `do_stuff()` | — |
| 2 | `return 1 # 3` | — | `return 1` | — |
| 3 | `do_more_stuff()<DEDENT> # 4` | — | `do_more_stuff()` | `<DEDENT>` (−1) |

> The `# 3` / `# 4` suffixes are source-line-number comments that Coconut embeds so
> `extract_line_num_from_comment` can recover the original source line for error
> reporting.

---

## Initial State

```
level               = 0
func_until_level    = None
disabled_until_level = None
last_terminator     = None
```

---

## Line-by-Line Trace

### Line 1 — `do_stuff()`

**Step 1 — apply leading indent (`ind_change(indent) = +1`)**

```
level = 0 + 1 = 1
```

**Step 2 — scope-exit checks** (both trackers are `None`, nothing to do)

**Step 3 — scope-entry checks**

- `def_regex.match("do_stuff()")` → no match → `func_until_level` unchanged
- `tco_disable_regex.match("do_stuff()")` → no match → `disabled_until_level` unchanged

**Step 4 — analysis gate** (`level == 1`, `disabled_until_level is None`, `base` non-empty → enter)

- `last_terminator is None` → **no error**
- `terminator_stmt_regex.match("do_stuff()")` → no match → `last_terminator = None`

**Step 5 — apply trailing dedent** (none on this line)

```
level = 1
```

**State after line 1:**
```
level = 1 | func_until_level = None | disabled_until_level = None | last_terminator = None
```

---

### Line 2 — `return 1`

**Step 1 — apply leading indent** (none)

```
level = 1  (unchanged)
```

**Step 2 — scope-exit checks** (both trackers are `None`, nothing to do)

**Step 3 — scope-entry checks**

- `def_regex.match("return 1")` → no match
- `tco_disable_regex.match("return 1")` → no match

**Step 4 — analysis gate** (`level == 1`, `disabled_until_level is None`, `base` non-empty → enter)

- `last_terminator is None` → **no error**
- `terminator_stmt_regex.match("return 1")` → **MATCH**, `group(1) = "return"`
  - `raw_ln = extract_line_num_from_comment(comment)` → source line `3`
  - `term_ln = self.adjust(3)` → adjusted source line number
  - `last_terminator = ("return", term_ln)` ← **terminator recorded**

**Step 5 — apply trailing dedent** (none)

```
level = 1
```

**State after line 2:**
```
level = 1 | func_until_level = None | disabled_until_level = None | last_terminator = ("return", 3)
```

---

### Line 3 — `do_more_stuff()`

**Step 1 — apply leading indent** (none)

```
level = 1  (unchanged)
```

**Step 2 — scope-exit checks** (both trackers are `None`, nothing to do)

**Step 3 — scope-entry checks**

- `def_regex.match("do_more_stuff()")` → no match
- `tco_disable_regex.match("do_more_stuff()")` → no match

**Step 4 — analysis gate** (`level == 1`, `disabled_until_level is None`, `base` non-empty → enter)

- `last_terminator = ("return", 3)` → **NOT None** → 🚨 **ERROR REPORTED**:
  ```
  CoconutStyleError: found unreachable code after return statement (line 3)
  ```
  - `last_terminator = None` (reset — only one error per terminator)
- `terminator_stmt_regex.match("do_more_stuff()")` → no match → `last_terminator = None`

**Step 5 — apply trailing dedent (`ind_change(dedent) = −1`)**

```
level = 1 - 1 = 0
```

**State after line 3:**
```
level = 0 | func_until_level = None | disabled_until_level = None | last_terminator = None
```

---

## Summary Table

| Line | `level` after indent | `last_terminator` entering gate | Gate action | `last_terminator` after gate |
|------|---------------------|---------------------------------|-------------|------------------------------|
| `do_stuff()` | 1 | `None` | no error; no match; reset to `None` | `None` |
| `return 1` | 1 | `None` | no error; **match** → record `("return", 3)` | `("return", 3)` |
| `do_more_stuff()` | 1 | `("return", 3)` | **🚨 error fired**; reset; no match | `None` |

---

## Control Flow Diagram

```
for each line in raw_lines:
│
├─ level += ind_change(indent)          ← adjust depth for leading indent markers
│
├─ [scope-exit] if level left func/disabled scope → clear tracker
│
├─ [scope-entry] if def/loop/try/with → set func_until_level or disabled_until_level
│
└─ if level == 1 AND not in suppressed scope AND line has content:
    │
    ├─ if last_terminator is set:
    │   └─ 🚨 report error ("unreachable code after <kwd>")
    │       last_terminator = None
    │
    └─ if line matches terminator regex (return/raise/break/continue):
    │   └─ last_terminator = (keyword, source_line_num)
    │
    └─ else (ordinary statement):
        └─ last_terminator = None   ← reset (not unconditionally unreachable after this)
    │
level += ind_change(dedent)             ← adjust depth for trailing dedent markers
```

---

## Why the Design Works

| Property | How it is achieved |
|----------|--------------------|
| Only fires at the function's **direct body** | `level == 1` guard |
| Ignores `return` inside nested functions | `func_until_level` suppresses the gate while inside a nested `def` |
| Ignores `return`/`break` inside loops/try | `disabled_until_level` suppresses the gate while inside `for`/`while`/`try`/`with` |
| Does not fire for `if`-branch returns | `if` statements are **not** in `terminator_stmt_regex`; they reset `last_terminator` to `None` |
| Reports at the **terminator's** source line | Line number is extracted from the Coconut-embedded comment on the terminator line itself |
| Reports only **once** per terminator | `last_terminator` is cleared immediately after the first error is emitted |
