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

## The Four State Variables

These four variables are the entire state machine. At any point during the loop, they
collectively answer the question: "should this line trigger an error, and are we in a
position where triggering would be correct?"

---

### `level` — current indentation depth

Starts at 0 (before entering the function body). Incremented by each `<INDENT>` marker,
decremented by each `<DEDENT>`. The gate only fires at `level == 1`, which corresponds
to statements written directly in the function body — not inside any nested block.

---

### `disabled_until_level` — gate suppression tracker

The variable the **gate actually checks** (`disabled_until_level is None`). While it is
not `None`, no unreachable-code analysis runs at all — the gate is skipped entirely.

Set to `level` when entering any of:
- a nested `def` or `class` (via `def_regex`)
- a `for`, `while`, `try`, or `with` block (via `tco_disable_regex`)

Cleared when `level` drops back to or below `disabled_until_level` (i.e., when we
exit the suppressed block).

**Why it is needed**: a `return` inside a loop body or nested function is not
unconditional — the loop may not execute, and the nested function's `return` belongs
to that function, not the outer one. Suppressing the gate for these blocks prevents
false positives.

---

### `func_until_level` — nested-def boundary tracker

Set to `level` whenever a nested `def` is seen, alongside `disabled_until_level`.
Cleared when `level` drops back to or below `func_until_level`.

**Why it is needed** — it exists solely to prevent a nested `def` *inside an already-
suppressed block* from overwriting `disabled_until_level` with a deeper (wrong) level.

Without it, consider:

```python
def f():
    def g():        # func_until_level = 1, disabled_until_level = 1
        def h():    # without func_until_level guard:
            pass    #   disabled_until_level = 2  ← WRONG (overwrites 1)
    x = 2           # disabled_until_level clears at 2, not 1
                    # → x = 2 is still suppressed → false negative
```

The `def` entry check uses `func_until_level is None` as its outer guard. Once we are
already inside a nested `def`, this guard blocks any further `def` from touching
`disabled_until_level`. Loops do not need their own equivalent tracker because their
entry check already uses `disabled_until_level is None` as its guard, which is already
false inside any suppressed block.

---

### `last_terminator` — pending terminator

Either `None` or a `(keyword_str, source_line_num)` tuple, e.g. `("return", 3)`.

- Set when the gate sees a `return`, `raise`, `break`, or `continue`.
- Triggers an error when the **next** line passes through the gate while it is non-`None`.
- Reset to `None` immediately after an error fires (one error per terminator).
- Also reset to `None` by any ordinary (non-terminator) statement, because an ordinary
  statement means the flow is no longer unconditionally terminated.

The line number stored is the **terminator's** source line, not the unreachable line's,
so the error message points back to the statement that caused the problem.

---

## Initial State

### The function

```python
def func():
    do_stuff()
    return 1
    do_more_stuff()  # ← should be flagged as unreachable
```
### The raw lines
| # | Internal line (simplified) | Indent marker | Body | Dedent marker |
|---|---------------------------|---------------|------|---------------|
| 1 | `<INDENT>do_stuff()` | `<INDENT>` (+1) | `do_stuff()` | — |
| 2 | `return 1 # 3` | — | `return 1` | — |
| 3 | `do_more_stuff()<DEDENT> # 4` | — | `do_more_stuff()` | `<DEDENT>` (−1) |

### The states

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

## The Three Gate Conditions

The gate is:

```python
if level == 1 and disabled_until_level is None and base and not is_blank(line):
```

All three conditions must be true. Here is why each one is necessary.

---

### Condition 1: `level == 1`

**What it means**: only analyze statements that are direct children of the function body.

**Why it is needed**: a terminator at a deeper level is conditional. For example, a
`return` inside an `if`-branch is only executed when that branch is taken — code after
the `if` block is still reachable. The detector does not do branch analysis, so it
simply ignores everything below level 1.

**What happens without it**: the detector would see `return` inside an `if`-branch at
level 2 and arm `last_terminator`. The next statement at level 1 would then be
incorrectly flagged as unreachable even though the `if` might not have been taken.

---

### Condition 2: `disabled_until_level is None`

**What it means**: only analyze when not inside a suppressed block (`def`, `for`,
`while`, `try`, `with`).

**Why it is needed**: these blocks introduce control flow that makes a terminator inside
them non-unconditional. A `return` inside a `for` loop only executes on some iterations;
a `return` inside a nested `def` belongs to that function entirely.

**What happens without it**: the detector would see `return i` inside a `for` loop and
arm `last_terminator`. The statement after the loop would be falsely flagged as
unreachable, even though the loop might run zero iterations.

Note that `level == 1` alone is not sufficient here. The `for` header line itself
(`for i in ...:`) is at level 1, so it passes condition 1. The scope-entry check
(setting `disabled_until_level`) happens in step 3, before the gate in step 4. So
`for i in ...:` correctly sets `disabled_until_level` first, and then condition 2
blocks the gate on that same line. Without condition 2, the gate would open even
while `disabled_until_level` is set — making the suppression mechanism useless.

---

### Condition 3: `base and not is_blank(line)`

**What it means**: skip blank lines and comment-only lines.

**Why it is needed**: blank lines and comments are not executable statements. A blank
line between a `return` and the next real statement should neither reset
`last_terminator` to `None` (hiding the unreachable code) nor falsely trigger an error.

**What happens without it**: a blank line after `return` would enter the gate, find
`last_terminator` set, and fire an error pointing at the blank line instead of the
actual unreachable statement. Or if it matched no terminator, it would reset
`last_terminator` to `None` and the real unreachable statement after it would go
undetected.

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
