# PR Draft: fix #599 — detect unreachable code after unconditional terminators

Adds detection of unreachable code after unconditional terminator statements (`return` and `raise`) inside function bodies.
In `--strict` mode this raises a `CoconutStyleError` and in non-strict mode it emits a `CoconutWarning`.

**Example** — this now raises/warns:
```python
def func():
    do_stuff()
    return x
    do_more_stuff()  # unreachable — flagged
```

### Changes

- **`coconut/compiler/grammar.py`**: Added `terminator_stmt_regex` matching the four terminator keywords.
- **`coconut/compiler/compiler.py`**: Added `detect_unreachable_code` method and called it from `proc_funcdef` just before `detect_is_gen`.
- **`coconut/tests/main_test.py`**: Added tests for both the strict-mode error and the non-strict warning.

### Design

The check is intentionally conservative to avoid false positives. It only flags a terminator when:

- it appears at the **direct top level** of a function body (indentation level 1), and
- it is followed by another non-blank statement at the same level.

Terminators inside nested `def`s, loops, `try`/`with` blocks, or `if`/`else` branches are not flagged. One diagnostic is emitted per terminator (not per unreachable line) to avoid error cascades.

### Limitations

These limitations are left for future work:

- The algorithm only detect unreachable code appears at the **direct top level** of a function body (indentation level 1)
- Terminators inside nested `def`s, loops, `try`/`with` blocks, or `if`/`else` branches are not flagged
-


The following cases are not detected and are left as future work:

- `if`/`else` where every branch terminates (requires branch-exhaustive analysis)
- `while True:` without a `break` (requires loop body scanning)
- `try`/`except` where all clauses terminate
- Unreachable code inside loop bodies after `break`/`continue`
- Module-level unreachable code
- NOQA suppression (requires bypassing the existing `has_noqa_comment` helper; `noqa_able=False` for now)
- Precise line number reporting when compiling with `-c` (falls back to the `def` line)
