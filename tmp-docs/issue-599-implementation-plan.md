# Issue #599 Implementation Plan

**Issue**: Known unreachable code in a function after a return should throw an error on `--strict`

**Example**:
```python
def func():
    do_stuff()
    return x
    do_more_stuff()  # unreachable — should be an error in strict mode
```

---

## Overview

When the compiler is in `--strict` mode (or warn mode), it should detect statements that appear at the top-level of a function body after an unconditional terminator (`return`, `raise`, `break`, `continue`). These statements are unreachable by definition, and the feature should report them as a `CoconutStyleError` (strict) or a warning (non-strict).

---

## Files to Modify

1. `coconut/compiler/grammar.py` — add `terminator_stmt_regex`
2. `coconut/compiler/compiler.py` — add detection logic and call site
3. `coconut/tests/main_test.py` — add test cases

---

## Step-by-Step Plan

### Step 1 — Add `terminator_stmt_regex` to `grammar.py`

In `grammar.py`, find the block where compiled regexes are defined as class attributes (near `return_regex`). Add a new regex that matches any unconditional terminator keyword at the start of a statement:

```python
# near the existing return_regex:
terminator_stmt_regex = compile_regex(r"\b(return|raise|break|continue)\b")
```

This regex is used at analysis time to identify lines that unconditionally terminate control flow.

**File**: `coconut/compiler/grammar.py`
**Location**: Near existing `return_regex` definition (around line 2812)

---

### Step 2 — Add `detect_unreachable_code` method to `compiler.py`

Add a new method on the `Compiler` class, alongside `detect_is_gen` and `transform_returns`, which are the existing methods that analyze function body lines.

The method receives the same `raw_lines` list that is used by `detect_is_gen` (the body of the function after the `def` line is removed).

**Algorithm**:

1. Track indentation `level` using `ind_change(indent)` and `ind_change(dedent)` from `split_leading_trailing_indent`.
2. Track two "suppression scopes":
   - `func_until_level`: inside a nested `def` — inner function terminators don't affect the outer body.
   - `disabled_until_level`: inside a `for`/`while`/`try`/`with` block — these blocks allow `return`/`break`/`continue` patterns that aren't globally unconditional.
3. At `level == 1` (direct body of the function), and outside suppressed scopes:
   - If a `last_terminator` is set and a non-blank, non-comment code line is encountered, raise/warn about unreachable code.
   - If the current line matches `terminator_stmt_regex`, record it as `last_terminator = (keyword, adjusted_source_ln)`.
   - Any other code line at level 1 resets `last_terminator` to `None` (e.g., an `if`-branch that only sometimes returns does not make everything after it unreachable).

```python
def detect_unreachable_code(self, original, loc, raw_lines):
    """Detect unreachable code after unconditional terminator statements (strict mode only)."""
    level = 0
    func_until_level = None
    disabled_until_level = None
    last_terminator = None  # (keyword_str, adjusted_source_ln) or None

    for line in normalize_indent_markers(list(raw_lines)):
        indent, body, dedent = split_leading_trailing_indent(line)
        base, comment = split_comment(body)

        level += ind_change(indent)

        # Leave inner function/class scope
        if func_until_level is not None and level <= func_until_level:
            func_until_level = None
        # Leave disabled flow-control block
        if disabled_until_level is not None and level <= disabled_until_level:
            disabled_until_level = None

        # Entering a nested def — inner terminators don't affect outer scope
        if func_until_level is None and self.def_regex.match(base):
            func_until_level = level
            if disabled_until_level is None:
                disabled_until_level = level

        # Entering a loop/try/with — disable checking inside
        if disabled_until_level is None and self.tco_disable_regex.match(base):
            disabled_until_level = level

        # Only analyze at top level of function body, outside suppressed scopes
        if level == 1 and disabled_until_level is None and base and not is_blank(line):
            if last_terminator is not None:
                term_kwd, term_ln = last_terminator
                self.strict_err_or_warn(
                    "found unreachable code after {kwd} statement".format(kwd=term_kwd),
                    original,
                    loc,
                    ln=term_ln,
                    noqa_able=False,
                    endpoint=False,
                )
                last_terminator = None  # Report only once per terminator

            m = self.terminator_stmt_regex.match(base)
            if m:
                raw_ln = extract_line_num_from_comment(comment)
                term_ln = self.adjust(raw_ln) if raw_ln is not None else None
                last_terminator = (m.group(1), term_ln)
            else:
                last_terminator = None  # Non-terminator resets tracking

        level += ind_change(dedent)
```

**File**: `coconut/compiler/compiler.py`
**Location**: After `detect_is_gen` method (around line 2614)

**Imports needed** (add to the existing import block from `coconut.compiler.util`):
- `is_blank`
- `extract_line_num_from_comment`

These are already available in `coconut/compiler/util.py`.

---

### Step 3 — Call `detect_unreachable_code` from `proc_funcdef`

Inside `proc_funcdef`, immediately after the function keyword detection loop (the loop that processes `addpattern`, `copyclosure`, `case`) and before `detect_is_gen`, add:

```python
# Detect unreachable code (strict/warn mode)
self.detect_unreachable_code(original, loc, raw_lines)
```

This ensures we check the actual function body (after the `def` line and any keyword prefix lines are removed from `raw_lines`).

**File**: `coconut/compiler/compiler.py`
**Location**: Inside `proc_funcdef`, just before the `detect_is_gen` call (around line 2869)

---

### Step 4 — Add Test Cases in `main_test.py`

Add two test methods to the `TestShell` class:

```python
def test_strict_unreachable_code_error(self):
    """--strict should raise an error for code after return."""
    call_coconut(
        ["--strict", "-c", "def f():\n    return 1\n    x = 2\n"],
        expect_retcode=1,
        check_errors=False,
        assert_output="found unreachable code after return statement",
    )

def test_strict_unreachable_code_warning(self):
    """Without --strict, unreachable code after return should warn."""
    call_coconut(
        ["-c", "def f():\n    return 1\n    x = 2\n"],
        check_errors=False,
        assert_output="found unreachable code after return statement",
    )
```

**File**: `coconut/tests/main_test.py`

---

## Edge Cases and Handling

| Case | Handling |
|------|----------|
| Nested `def`/`class` inside function | Suppressed via `func_until_level` — inner `return` does not flag outer code |
| `for`/`while`/`try`/`with` blocks | Suppressed via `disabled_until_level` — `break`/`continue`/`return` inside loops are intentional |
| `if`/`else` with returns in all branches | **Not detected** (by design) — the check is intentionally conservative; only unconditional top-level terminators are flagged |
| `raise` statement | Flagged as a terminator (same as `return`) |
| `break`/`continue` at top of function body | Rare, but flagged — they are unconditional terminators at that level |
| Pattern-matched `match def` functions | Bodies are also processed by `proc_funcdef`; generated code lines with `ln=None` are silently skipped |
| Statement lambdas (`is_stmt_lambda=True`) | Also processed by `proc_funcdef`; detection applies correctly |
| `pass` statement | **Not treated as a terminator** — `pass` does not prevent subsequent code from executing |
| NOQA comments | `noqa_able=False` initially; can be added as a follow-up once the correct `loc` for body lines is accessible |

---

## Error Message

```
CoconutStyleError: found unreachable code after return statement (line N)
```

In warn-only mode (non-strict):
```
CoconutWarning: found unreachable code after return statement (line N)
```

---

## Testing Strategy

1. **Positive tests** (should error in strict mode): unreachable code after `return`, `raise`, `break`, `continue` at function top level.
2. **Negative tests** (should NOT error):
   - `return` inside a nested `def`
   - `return` inside a `for`/`while` loop
   - `if/else` where both branches return but there's code after
   - Empty function body
3. **Compile existing test suite** (`make test`) — confirms no false positives in the Coconut test suite itself.

---

## Implementation Order

1. Add `terminator_stmt_regex` to `grammar.py` (~5 min)
2. Add imports (`is_blank`, `extract_line_num_from_comment`) in `compiler.py` (~2 min)
3. Add `detect_unreachable_code` method to `compiler.py` (~20 min)
4. Call `detect_unreachable_code` from `proc_funcdef` (~2 min)
5. Add test cases to `main_test.py` (~10 min)
6. Run `make test` to verify correctness and no false positives

---

## Key Precedents in the Codebase

- `detect_is_gen` (`compiler.py`) — iterates `raw_lines` in the same way; use as structural template
- `transform_returns` (`compiler.py`) — another `raw_lines` processor; same line-by-line pattern
- `check_undefined_name` (`compiler.py`) — uses `strict_err_or_warn` with `ln=outer_ln` — exact pattern to follow for error reporting
- `return_regex` (`grammar.py`) — existing regex for `return`; `terminator_stmt_regex` follows the same pattern
