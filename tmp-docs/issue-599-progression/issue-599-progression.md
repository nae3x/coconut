# Issue #599 — Progression Report

**Issue**: [evhub/coconut#599](https://github.com/evhub/coconut/issues/599) — Known unreachable code in a function after a return should throw an error on `--strict`

**Branch**: `fix-599-yield-error` (based on `master`)

**Period**: ~2 weeks

---

## Phase 1: Issue Analysis and Information Gathering

### The request

The issue asks for Coconut's `--strict` mode to detect and report unreachable code after unconditional terminator statements (`return`, `raise`, `break`, `continue`). For example:

```coconut
def func():
    do_stuff()
    return x
    do_more_stuff()  # unreachable — should error in --strict, warn otherwise
```

This is a standard feature in most compiled languages (GCC, Clang, rustc, javac all have it), but Coconut did not have it.

### Understanding the Coconut compiler

The first task was understanding where this check could fit in Coconut's three-phase compilation pipeline:

```
Phase 1: PRE-PROCESSING    Phase 2: PYPARSING         Phase 3: POST-PROCESSING
(compiler.py - pre())      (grammar.py)               (compiler.py - post())
```

This required studying how each phase transforms the source code, what information is available at each stage, and what would be destroyed by subsequent stages.

---

## Phase 2: Where to Implement the Fix

**Document**: [issue-599-why-detection-at-post-processing.md](01-why-detection-at-post-processing.md)

The detection needs four things simultaneously:
1. **Function body boundaries** — know where a function starts and ends
2. **Nesting-level tracking** — distinguish top-level `return` from one inside an `if` branch
3. **Statement identification** — recognize `return`, `raise`, `break`, `continue`
4. **Source line numbers** — map issues back to original Coconut source lines

### Why not pre-processing?

Pre-processing operates on raw text. It has no understanding of function boundaries (Coconut has many `def` variants: `async def`, `addpattern def`, `copyclosure def`, `match def`, etc.). After `ind_proc()`, indentation becomes `openindent`/`closeindent` markers, but these encode *depth*, not *kind* — you can't tell if an indent opens an `if` block or a nested `def`. Building this understanding would require writing a second, simplified Coconut parser.

### Why not during PyParsing?

Two architectural blockers:
- **No AST**: Coconut doesn't build an Abstract Syntax Tree. Parse actions consume tokens and immediately emit Python strings. There's no tree to walk.
- **Function bodies are deferred**: When PyParsing encounters `def`, the handler wraps the entire function body in a deferred marker (`_coconut_func_wrapper42`). The actual body is stored in a reference table for later processing. It is *not available* as a complete unit during parsing.

### Why post-processing works

Post-processing begins with `deferred_code_proc()`, which resolves deferred function markers by calling `proc_funcdef()`. At this point:
- Function bodies are available as bounded `raw_lines` lists
- `openindent`/`closeindent` markers are still present (not yet resolved to whitespace)
- Line-number comments are still intact (not yet resolved to final form)
- Two existing methods (`detect_is_gen` and `transform_returns`) already iterate `raw_lines` using the exact same pattern

The detection was placed inside `proc_funcdef()`, just before `detect_is_gen()` — the **unique moment** where all four prerequisites converge.

---

## Phase 3: Implementation Plan

**Document**: [issue-599-implementation-plan.md](02-implementation-plan.md)

### Files modified

1. **`coconut/compiler/grammar.py`** — Added `terminator_stmt_regex = compile_regex(r"\b(return|raise|break|continue)\b")` as a class attribute, following the pattern of existing regexes like `return_regex`.

2. **`coconut/compiler/compiler.py`** — Added the `detect_unreachable_code` method and its call site in `proc_funcdef`.

3. **`coconut/tests/main_test.py`** — Added test cases for both strict-mode error and non-strict warning.

### Algorithm

The algorithm is a single-pass, flat-state loop over `raw_lines`:

- Track `level` using `ind_change()` on `openindent`/`closeindent` markers
- Track two suppression scopes:
  - `func_until_level`: inside a nested `def` — inner terminators don't affect the outer scope
  - `disabled_until_level`: inside `for`/`while`/`try`/`with` — these allow `return`/`break`/`continue` patterns that aren't globally unconditional
- At `level == 1` (direct function body), outside suppressed scopes:
  - If `last_terminator` is set and a non-blank line appears → report unreachable code
  - If the line matches `terminator_stmt_regex` → record as `last_terminator`
  - Otherwise → reset `last_terminator` (an intervening statement means the flow isn't unconditionally terminated)

This follows the exact same iteration pattern used by `detect_is_gen` and `transform_returns`, making it a natural peer in the codebase.

### Commit

**`b259d43c`** — *Apply fix for issue 599*

---

## Phase 4: Performance Analysis

**Document**: [issue-599-performance-analysis.md](03-performance-analysis.md)

The performance impact was analyzed to ensure the new pass wouldn't slow down compilation. Key findings:

- **Per-line cost**: String splitting, precompiled regex `.match()`, integer comparisons — all trivially fast. No memory allocation, no parsing, no backtracking.
- **Relative to existing passes**: Cheaper than `transform_returns` (which additionally runs `cached_match_in` and `post_transform` parser calls), comparable to `detect_is_gen`.
- **Relative to PyParsing**: PyParsing (the dominant cost) performs tens of thousands of rule match attempts with backtracking. The ~4,200 trivial operations from `detect_unreachable_code` (for a 20-function file) are well under 0.1% of total compilation cost.
- **Scope**: Per-function body (typically 10-100 lines), not per-file.

**Verdict**: Negligible impact. Like adding a 1-second detour to a 10-hour road trip.

---

## Phase 5: Comprehensive Test Cases as Documentation

**Document**: [issue-599-more-test-case-impl-plan.md](04-more-test-case-impl-plan.md)

The initial implementation shipped with only two tests: one for `--strict` error and one for non-strict warning, both testing the same `return` + code scenario. This left the detection's behavioral boundaries — what it intentionally skips and why — undocumented through tests. A contributor reading the test file would have no way to understand the scope of the detector.

### Goal

Add tests that serve as **executable documentation** of the detection scope, covering all terminator types and all suppression mechanisms.

### Positive tests (should detect unreachable code)

Two additional tests for `raise` as a terminator keyword (with and without arguments), verifying the detector handles more than just `return`. No `break`/`continue` positive tests were added — and this is where we realized that `break` and `continue` **cannot actually trigger the detector**. Since the detection only operates at level 1 (the direct top level of a function body), and `break`/`continue` are only valid inside a loop (which is at level 2+), the compiler would reject `break`/`continue` at level 1 as a syntax error before `detect_unreachable_code` ever runs. In practice, only `return` and `raise` are meaningful terminators for this detection.

Based on this realization, we updated `terminator_stmt_regex` from `r"\b(return|raise|break|continue)\b"` to `r"\b(return|raise)\b"`, removing the two keywords that can never match at the detection level. This keeps the regex simpler and more honest about what the detector actually does.

### Negative tests (should NOT detect unreachable code)

Nine tests documenting each suppression boundary:

| Test | What it documents |
|---|---|
| Nested `def` with inner `return` | `func_until_level` suppression — inner function terminators don't affect the outer body |
| `for` loop with `return` inside | `disabled_until_level` suppression for `for` |
| `while` loop with `return` inside | `disabled_until_level` suppression for `while` |
| `try` block with `return` inside | `disabled_until_level` suppression for `try` |
| `with` block with `return` inside | `disabled_until_level` suppression for `with` |
| `if` without `else`, `return` inside | `return` at level 2 never sets `last_terminator` at level 1 |
| `if`/`else` both returning | Conservative design — no branch-exhaustive analysis, so NOT detected |
| Function ending with `return` only | `last_terminator` set but no subsequent line — no false positive |
| Empty function (`pass` only) | `pass` is not a terminator — baseline sanity check |

All negative tests use `--strict` mode, so any false positive would become a hard error that fails the test.

---

## Phase 6: End-to-End Test Failures — The `yield` Problem

**Document**: [issue-599-e2e-fail-report.md](05-e2e-fail-report.md)

Running `make test-univ` revealed two categories of test failures:

### Failure 1: User-written `yield` after `return` (idiomatic pattern)

Three test functions used `return` followed by `yield None` — a well-known Python idiom to make a function a generator:

```coconut
def it_ret(x):
    return x
    yield None    # unreachable, but intentional — makes this a generator
```

The checker correctly identifies `yield None` as unreachable. **Fix**: Rewrote these test functions to use `if False: yield None` before the `return` instead — same generator semantics, no unreachable code.

### Failure 2: Compiler-generated `yield` from `yield def`

Coconut's `yield def` syntax creates implicit generators. The compiler (in `keyword_funcdef_handle`) **appends** `if False: yield` at the *bottom* of the function body. For math-def forms like `yield def f(x) = x`, the body compiles to `return x` followed by `if False: yield` — triggering a false positive.

**Root cause**: `keyword_funcdef_handle` appends to the end, creating unreachable code after any terminating statement.

---

## Phase 7: First Fix Attempt — Prepend `yield` (v1)

**Document**: [issue-599-fix-yield-side-effect.md](06-fix-yield-side-effect.md)

### Approach

Changed `keyword_funcdef_handle` to **prepend** the `if False: yield` block at the *start* of the function body instead of appending at the end. The insertion finds the first `openindent` marker (body start) and inserts after it:

```python
if kwd == "yield":
    if_false_yield = handle_indentation("""
if False:
    yield
    """, add_newline=True)
    idx = funcdef.index(openindent) + 1
    funcdef = funcdef[:idx] + if_false_yield + funcdef[idx:]
```

### Commit

**`ab2c5e71`** — *Fix case detect unreachable code when yield*

### Result

Tests passed. But this introduced an unintended side effect discovered during further analysis.

---

## Phase 8: Unintended Side Effect — Broken Docstrings (v2)

**Document**: [issue-599-fix-yield-side-effect-v2.md](07-fix-yield-side-effect-v2.md)

### The problem

Prepending `if False: yield` at the top of the function body pushes any docstring down. Python's PEP 257 requires the docstring to be the **first statement** in the body. With `if False: yield` before it, `__doc__` is not set and static analysis tools don't recognize the docstring.

```coconut
yield def f(x):
    """This is a docstring."""
    return x
```

Compiles to:
```python
def f(x):
    if False:        # ← now the first statement
        yield
    """This is a docstring."""    # ← no longer recognized as docstring
    return x
```

### The better fix

Instead of moving the yield insertion, keep it at the original bottom position (proven behavior) and make the **detector** smarter:

1. **Revert** `keyword_funcdef_handle` to the original append-at-bottom logic.
2. **Modify** `detect_unreachable_code` to skip lines that lack a source line number comment — these are compiler-generated code (like `if False: yield`) that was never written by the user.

```python
if last_terminator is not None:
    # Skip compiler-generated code (no source line number = not user code)
    raw_ln = extract_line_num_from_comment(comment)
    if raw_ln is not None:
        term_kwd, term_ln = last_terminator
        self.strict_err_or_warn(...)
    last_terminator = None
```

This separates concerns cleanly: yield insertion stays simple, and the detector learns to distinguish user code from compiler-generated code.

### Test added

A dedicated test (`test_strict_unreachable_code_yield_def`) verifies that `yield def f(x) = x` compiles without triggering a false positive in `--strict` mode. This directly exercises the "skip lines without source line number" logic — without it, a future refactor could regress the v2 fix with no test catching it.

### Commit

**`1e3bd9ba`** — *Revert yield insertion to bottom; skip unreachable warning for compiler-generated code*

### Result

All tests pass. Docstrings preserved. No false positives on the existing test suite.

---

## Phase 9: Limitations and Future Work

**Document**: [issue-599-future-works.md](08-future-works.md)

The implementation is intentionally **conservative**: zero false positives, many false negatives. The following cases are not detected and are documented as future work:

| Enhancement | Complexity | Notes |
|---|---|---|
| Branch-exhaustive `if`/`else` analysis | Medium-High | Highest impact; would catch the most common missed pattern |
| `try`/`except` where all paths terminate | High | Similar to `if`/`else` but with 4 clause types |
| `while True` without `break` | Low-Medium | Uncommon pattern |
| `break`/`continue` inside loop bodies | Low-Medium | Requires per-level terminator tracking |
| Module-level unreachable code | Medium | Needs a new entry point outside `proc_funcdef` |
| NOQA suppression | Low-Medium | Cannot use existing `has_noqa_comment` (wrong `loc`); requires direct `self.comments` lookup |
| Precise line numbers in `-c` mode | Medium | Falls back to `def` line when no line comments exist |

The fundamental constraint is Coconut's **no-AST architecture**. All improvements must work within the line-by-line scanning loop. A proper control-flow graph (as used by GCC/Clang/rustc) is not available.

**Document**: [issue-599-not-support-NOQA.md](09-not-support-NOQA.md) — Detailed explanation of why the existing `has_noqa_comment` mechanism is incompatible with the unreachable code checker (the `loc` parameter points to the `def` line, not the unreachable line).

---

## Commit History

```
1e3bd9ba Revert yield insertion to bottom; skip unreachable warning for compiler-generated code
ab2c5e71 Fix case detect unreachable code when yield
80fc7af9 Remove unused documents
9dc3b36e Merge branch 'documents' into fix-599
b259d43c Apply fix for issue 599
```

---

## Summary

| Phase | What happened | Key decision |
|---|---|---|
| 1. Analysis | Studied the issue and the Coconut compiler pipeline | — |
| 2. Where | Evaluated all three phases; chose post-processing | Only place where function boundaries, nesting markers, and line numbers coexist |
| 3. Implementation | Single-pass line scanner in `proc_funcdef`, following `detect_is_gen` pattern | Conservative: only top-level unconditional terminators |
| 4. Performance | Analyzed per-line cost vs. PyParsing | Negligible (<0.1% of total compilation) |
| 5. More tests | Added 11 tests as executable documentation of detection scope | Cover all terminator types and suppression boundaries |
| 6. Test failures | `yield` after `return` (user idiom + compiler-generated) | Two separate root causes, two separate fixes needed |
| 7. Fix v1 | Prepend `if False: yield` at body top | Tests pass, but... |
| 8. Fix v2 | Revert prepend; teach detector to skip compiler-generated code | Clean separation of concerns; docstrings preserved |
| 9. Future work | Documented 7+ enhancements as future work | Keep scope focused; ship what works |
