# Issue #599 Implementation Log

**Issue**: Known unreachable code in a function after a return should throw an error on `--strict`
**Date**: 2026-03-01

---

## Changes Made

### 1. `coconut/compiler/grammar.py` (line 2813)

Added `terminator_stmt_regex` — a compiled regex matching `return|raise|break|continue` keywords. Placed alongside the existing `return_regex` and `tco_disable_regex`.

### 2. `coconut/compiler/compiler.py`

**Imports** (line 218): Added `is_blank` to the imports from `coconut.compiler.util`.

**New method** `detect_unreachable_code` (lines 2617–2671): Added after `detect_is_gen`, following the same structural pattern (iterate `raw_lines` with `normalize_indent_markers`, track `level`, `func_until_level`, `disabled_until_level`). At level 1 (top of function body), if a terminator statement is seen and then a non-blank code line follows, it calls `self.strict_err_or_warn(...)`.

**Call site** in `proc_funcdef` (line 2923): Added `self.detect_unreachable_code(original, loc, raw_lines)` just before the existing `detect_is_gen` call.

### 3. `coconut/tests/main_test.py` (lines 915–929)

Added two test methods to `TestShell`:
- `test_strict_unreachable_code`: Verifies `--strict` mode raises `CoconutStyleError` (exit code 1) for code after `return`.
- `test_unreachable_code_warning`: Verifies non-strict mode produces a `CoconutSyntaxWarning` (exit code 0).

---

## Problem Encountered: Line Number Extraction

**Problem**: The implementation plan suggested using `extract_line_num_from_comment(body)` to get line numbers from comments. However, at the stage where `proc_funcdef` processes function bodies, line numbers are not stored as standard `# N` comments. Instead, they use Coconut's internal `lnwrapper` format: `‡<number>⏹` (double-dagger + number + stop-square unicode markers).

The `extract_line_num_from_comment` function splits on `#` to find line numbers, so it returns `None` for these internal markers.

**Fix**: Replaced the `extract_line_num_from_comment` call with direct parsing of the `lnwrapper`/`unwrapper` format:
```python
if lnwrapper in comment:
    for ln_str in comment.split(lnwrapper)[1:]:
        if ln_str.endswith(unwrapper):
            term_ln = self.adjust(int(ln_str[:-1]))
```
This mirrors the approach used in `post_proc_line` (around line 2407 of compiler.py). As a result, `extract_line_num_from_comment` was removed from the imports since it's no longer used.

---

## Problem Encountered: Test `assert_output` Matching

**Problem**: The initial test cases failed because `call()` defaults to `assert_output_only_at_end=True`, which checks only the **last line** of output. But the error/warning message appears on an earlier line (the last line is the source code context like `  def f():`).

**Fix**: Added `assert_output_only_at_end=False` to both test calls so the assertion checks the full output.

---

## Verification Results

### Smoke Tests (all passed)
- `--strict` with `return` + unreachable code → `CoconutStyleError` (exit code 1), correct line number
- `--strict` with `raise` + unreachable code → `CoconutStyleError` (exit code 1)
- Non-strict with unreachable code → `CoconutSyntaxWarning` (exit code 0)
- `return` inside `if` branch (not unconditional) → No warning (correct)
- `return` inside `for` loop → No warning (correct)
- `return` inside nested `def` → No warning (correct)
- `return` inside `try` block → No warning (correct)
- Multiple functions, only one with unreachable code → Correct function flagged

### Compilation of All Test Files (no false positives)
All 8 agnostic `.coco` test files compiled successfully with `--strict`:
- `__init__.coco`, `__main__.coco`, `main.coco`, `primary_1.coco`, `primary_2.coco`, `specific.coco`, `suite.coco`, `tutorial.coco`, `util.coco`

### pytest `TestShell` Results
- 15 passed (including both new tests)
- 2 failed (pre-existing failures: `test_find_packages` and `test_import_hook` — stdin capture issues unrelated to our changes)

### Full `make test`
Not fully completed (compilation takes 20+ minutes with `--force --no-cache`). The individual file compilations above confirm no false positives.

---

## Summary

The implementation follows the plan closely with two deviations:
1. Line number extraction uses direct `lnwrapper`/`unwrapper` parsing instead of `extract_line_num_from_comment`
2. Tests use `assert_output_only_at_end=False`

Both deviations are minor and stem from details of the internal representation that weren't visible during planning.
