# Issue 599: Fix `keyword_funcdef_handle` Side Effect — v2

> Supersedes [issue-599-fix-yield-side-effect.md](06-fix-yield-side-effect.md), which proposed prepending `if False: yield` at the top of the function body. That approach breaks PEP 257 docstrings (`__doc__` is not set because `if False: yield` becomes the first statement instead of the docstring).

## Problem

The v1 fix (commit `ab2c5e71`) moved the `if False: yield` block from the **bottom** to the **top** of function bodies in `yield def` to avoid the unreachable code checker flagging it. However, this breaks PEP 257 docstrings — the `if False: yield` block now precedes the docstring, so `__doc__` is not set and static tools don't recognize it.

## Plan

**Revert** the yield insertion to the original append-at-bottom, and instead make `detect_unreachable_code` **skip lines that lack a source line number comment** (i.e., compiler-generated code).

This keeps the two concerns separated: yield insertion stays simple (original proven behavior), and the detector becomes smarter about what it flags.

## Changes

### 1. Revert `keyword_funcdef_handle` — `coconut/compiler/compiler.py:5320-5332`

Revert to the original append-at-bottom with `extra_indent=1`:

```python
if kwd == "yield":
    funcdef += handle_indentation(
        """
if False:
    yield
        """,
        add_newline=True,
        extra_indent=1,
    )
```

Remove the `idx`/insert logic and the comment about the unreachable code checker.

### 2. Modify `detect_unreachable_code` — `coconut/compiler/compiler.py:2651-2661`

Before firing the warning, check if the unreachable line has a source line number. If it doesn't, it's compiler-generated — skip the warning:

```python
if last_terminator is not None:
    # Skip compiler-generated code (e.g. `if False: yield` from `yield def`)
    # which has no source line number comment
    raw_ln = extract_line_num_from_comment(comment)
    if raw_ln is not None:
        term_kwd, term_ln = last_terminator
        self.strict_err_or_warn(
            "found unreachable code after " + term_kwd + " statement",
            original,
            loc,
            ln=term_ln,
            noqa_able=False,
            endpoint=False,
        )
    last_terminator = None
```

### 3. Do NOT revert test changes

The test changes in `target_sys_test.coco` (commit `ab2c5e71`) rewrote user-written `it_ret`, `it_ret_none`, `it_ret_tuple` to use `if False: yield None` before `return`. These are **user-written** functions (not `yield def`), so the detector would still correctly flag them. Those changes must stay.

## Side-effects

### 1. External Python linters still see `if False: yield` after `return`

Tools like `pylint`, `pyflakes`, or `mypy` that analyze the *compiled `.py` output* would still see `if False: yield` after a `return`. This fix only addresses Coconut's own `--strict` checker. However, this was the original behavior before the prepend fix, so it is not a regression. The `if False:` pattern is a well-known Python idiom that some linters already special-case.

### 2. Theoretical false negatives in the detector

By skipping lines without a source line number, the detector would miss any unreachable code that happens to lack a line number comment. In practice, every line of user code gets a line number comment during compilation — but if there were a bug in line number tracking, this fix would silently hide it.

### 3. `last_terminator` state reset

When the detector encounters compiler-generated unreachable code, it resets its tracking state (`last_terminator = None`). If user code somehow appeared *after* the `if False: yield` block, it wouldn't be flagged. In practice this can't happen — the block is appended at the very end and nothing adds user code after it.

### What's NOT a side-effect

- Docstrings are fully preserved (yield is at the bottom, after everything)
- Runtime behavior is identical (the original append-at-bottom was the proven working approach)
- The fix generalizes to any compiler-generated unreachable code, not just yield

## Files to modify

- `coconut/compiler/compiler.py` (2 locations)

## Verification

```bash
make test-univ    # Recompile and run universal tests — confirms no regressions and no false positives
```

Also manually verify docstring preservation:
```bash
python -m coconut -c "yield def f(x):
    \"\"\"Docstring.\"\"\"
    return x
" | grep -A5 "def f"
```

Expected: `"""Docstring."""` appears as the first statement in the body (before `return x`), with `if False: yield` appended after.
