# Issue #599 Implementation Log

## Summary

Implemented unreachable code detection for Coconut's `--strict` mode. When code appears after an unconditional terminator (`return`, `raise`, `break`, `continue`) at the top level of a function body, the compiler now raises a `CoconutStyleError` in strict mode or a `CoconutSyntaxWarning` otherwise.

---

## Changes Made

### 1. `coconut/compiler/grammar.py`
- **Added `terminator_stmt_regex`** (line ~2813): A compiled regex matching `return|raise|break|continue` at word boundaries. Placed alongside the existing `return_regex`.

### 2. `coconut/compiler/compiler.py`
- **Added imports**: `is_blank` and `extract_line_num_from_comment` from `coconut.compiler.util` (line ~198).
- **Added `detect_unreachable_code` method** (after `detect_is_gen`, line ~2618): A new method that iterates `raw_lines` (the function body) and tracks indentation level, nested function scopes (`func_until_level`), and flow-control blocks (`disabled_until_level`). At level 1 (direct body), it detects code after unconditional terminators.
- **Called `detect_unreachable_code` from `proc_funcdef`** (line ~2923): Inserted just before the `detect_is_gen` call, so it runs on every function definition.

### 3. `coconut/tests/main_test.py`
- **Added `test_strict_unreachable_code_error`**: Verifies `--strict` mode raises an error (retcode=1) for code after `return`.
- **Added `test_strict_unreachable_code_warning`**: Verifies non-strict mode emits a warning (retcode=0) for the same code.
- Tests ended up in `TestCompilation` class (not `TestShell` as planned) — this is fine since they test compilation behavior and don't depend on test file compilation infrastructure.

---

## Problems Found

### 1. Test placement — plan said `TestShell`, actual location is `TestCompilation`

The plan suggested adding tests to `TestShell`, but the `COCONUT_TEST_VERBOSE` / `COCONUT_TEST_TRACE` blocks (used as placement anchors) are actually inside `TestCompilation`, not `TestShell`. The tests were placed before those blocks, making them part of `TestCompilation`. This is functionally correct — both classes use `call_coconut()` the same way.

### 2. `assert_output_only_at_end` behavior

The plan's test for `test_strict_unreachable_code_error` did not include `assert_output_only_at_end=False`. By default, the `call()` function checks `assert_output` against only the **last non-ignored line** of output. When a `CoconutStyleError` is raised, the output looks like:

```
CoconutStyleError: found unreachable code after return statement (...) (line N)
  def f():
```

The last line is the snippet (`def f():`), not the error message. So `assert_output_only_at_end=True` (default) fails to find the expected string. Both tests needed `assert_output_only_at_end=False`.

### 3. Line number points to function definition, not the terminator

The error message reports the line of the `def` statement, not the `return`/`raise` line itself. This happens because:
- `extract_line_num_from_comment` returns `None` for `-c` input (no line number comments in the compiled output at that stage).
- When `ln=None`, `make_err` falls back to `self.outer_ln` or `lineno(loc, original)`, which points to the function definition's location in the parsed original string.
- The snippet also shows `def f():` for the same reason — it's derived from `loc`.

This is the same behavior as `check_undefined_name` and is acceptable. When compiling from `.coco` files with line number comments embedded, `extract_line_num_from_comment` may return a more precise line number. This could be improved in a follow-up.

### 4. `noqa_able` left as `False`

The plan suggested `noqa_able=False` initially. This means users cannot suppress the warning with `# NOQA`. This is intentional because at the point `detect_unreachable_code` runs, we're operating on the compiled/intermediate `raw_lines` (not the original source), so NOQA comments from source may not be reliably accessible. A follow-up could add NOQA support if needed.

---

## What the Plan Did Not Account For

1. **The `assert_output_only_at_end` issue** — the plan's test code would have failed without this fix. The `call()` helper's default behavior of checking only the last output line was not considered.

2. **Test class mismatch** — the plan assumed the `COCONUT_TEST_VERBOSE` block was inside `TestShell`, but it's inside `TestCompilation`. The indentation structure of nested `if TEST_ALL:` / `if not PYPY:` blocks made this non-obvious.

3. **`make test` duration** — the plan estimated ~40 min total implementation time but did not account for `make test` taking potentially 1+ hour due to mypy type-checking. A faster verification approach (compile-only with `--strict`, then pytest on just the new tests) was used instead.

---

## Verification Results

| Check | Result |
|-------|--------|
| `--strict -c "def f(): return 1; x = 2"` raises error | Pass |
| `-c "def f(): return 1; x = 2"` emits warning | Pass |
| `raise` triggers detection | Pass |
| Nested `def` does NOT trigger false positive | Pass |
| `return` inside `for` loop does NOT trigger false positive | Pass |
| Compile full test suite with `--strict` | Pass (no false positives) |
| `pytest -k unreachable` | 2/2 pass |
| Full `make test` (compile + run tests, no mypy) | Pending |
