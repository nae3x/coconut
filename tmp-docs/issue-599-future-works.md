# Issue #599 — Future Work for Unreachable Code Detection

## Current State

The current `detect_unreachable_code` implementation is intentionally **conservative**: it only flags code that is *trivially provable* as unreachable — an unconditional terminator (`return`, `raise`, `break`, `continue`) followed by another statement, both at the **direct top level** (level 1) of a function body. Anything inside a nested scope (loops, `try`, `with`, `if`, inner `def`) is completely ignored.

This design was chosen because it produces **zero false positives** in exchange for **many false negatives** (unreachable code that goes undetected). The remainder of this document catalogs those false negatives, explains why each one would require increased algorithmic complexity to handle, and discusses the trade-offs involved.

---

## 1. Branch-Exhaustive `if`/`elif`/`else` Analysis

### What is missed

```python
def f(x):
    if x > 0:
        return 1
    else:
        return -1
    print("unreachable")  # NOT detected
```

Every branch terminates, so `print("unreachable")` can never execute. The current algorithm does not detect this because `if` is not in `tco_disable_regex` (so it doesn't suppress analysis), but the `return` statements are at level 2, not level 1. When the algorithm returns to level 1 after the `else` block, `last_terminator` is `None` — it was never set at level 1 in the first place.

### What would be needed

A branch-exhaustive analysis requires the algorithm to:

1. **Recognize `if`/`elif`/`else` chains as a unit**: The algorithm currently processes lines one at a time. It would need to track entry into an `if` block, accumulate termination status across `elif`/`else` branches, and only conclude "all branches terminate" when the final `else` is processed. Without an `else`, the `if` chain is not exhaustive by definition.

2. **Track per-branch termination state**: A new data structure — e.g., `branch_terminates: list[bool]` — would track whether each branch of the current `if`/`elif`/`else` chain ends with a terminator. When the chain ends (dedent back to the `if` level), the algorithm checks: did *every* branch terminate? If so, set `last_terminator` at the outer level.

3. **Handle nesting**: An `if` inside an `if` inside a function body creates a recursive problem. Each nesting level needs its own branch-tracking state. This effectively requires a stack:

   ```python
   # Conceptual: stack of branch-tracking contexts
   branch_stack: list[BranchContext]

   @dataclass
   class BranchContext:
       level: int
       has_else: bool
       all_branches_terminate: bool
   ```

4. **Handle `elif` correctly**: `elif` is syntactically a new branch at the same level as `if`. The algorithm must recognize `elif` as continuing the current `if` chain rather than starting a new one. At the raw-lines level, `elif` appears as a dedent (closing the previous branch) followed by `elif ...` and an indent (opening the new branch) — potentially on the same line due to Coconut's indent marker encoding.

### Complexity impact

The current algorithm is a single-pass, flat-state loop with O(n) time and O(1) space (excluding the input). Branch analysis would add a stack proportional to nesting depth, making it O(n) time and O(d) space where d is nesting depth. The implementation complexity roughly doubles: every scope transition must now check whether it's part of an `if` chain and update branch tracking accordingly.

### Risk of false positives

The main risk is Coconut-specific constructs that compile into `if`/`elif`/`else` chains but don't behave like them:

- **Pattern matching** (`match`/`case`): Coconut's `match` blocks compile into `if`/`elif` chains. A non-exhaustive match (no default case) would produce an `if`/`elif` without `else`, which should NOT be treated as exhaustive. Getting this wrong would produce false positives.
- **Ternary expressions**: Coconut compiles some expressions into `if`/`else` blocks. These are at the expression level, not the statement level, but at the raw-lines stage, the distinction may be lost.

---

## 2. `while True` Without `break`

### What is missed

```python
def f():
    while True:
        do_work()
    print("unreachable")  # NOT detected
```

A `while True` loop without a `break` is an infinite loop. Code after it is unreachable. The current algorithm treats all `while` loops the same (`tco_disable_regex` suppresses analysis inside them), and never examines what happens *after* the loop.

### What would be needed

1. **Detect `while True` vs `while <expr>`**: A regex or parse check to distinguish `while True:` (always infinite) from `while condition:` (may exit normally).

2. **Scan the loop body for `break`**: After recognizing `while True`, the algorithm must scan the loop body to determine if any `break` statement exists (at any nesting level within the loop, except inside nested functions). If there is no `break`, the loop never exits, and code after it is unreachable.

3. **Handle conditional `break`**: A `break` inside an `if` within the loop is still a `break` — it *can* cause the loop to exit. Only the *complete absence* of `break` in the loop body guarantees infinite iteration. This is conservative: a `break` in a dead branch (e.g., `if False: break`) would prevent the detection, but that's an acceptable false negative.

### Complexity impact

Moderate. It requires a sub-scan of loop bodies, which is conceptually similar to what `detect_is_gen` already does (scanning raw lines for `yield`). The main challenge is correctly scoping the `break` search to the current loop and not inner functions.

---

## 3. `try`/`except` Where All Paths Terminate

### What is missed

```python
def f():
    try:
        return compute()
    except Exception:
        return fallback()
    print("unreachable")  # NOT detected
```

Both the `try` body and the `except` handler terminate, so `print` is unreachable. The current algorithm suppresses all analysis inside `try` blocks.

### What would be needed

This is similar to branch-exhaustive `if`/`else` analysis, but harder:

1. **Track termination across `try`/`except`/`else`/`finally` clauses**: All four clause types need to be handled. The rules are complex:
   - If `try` + all `except` clauses terminate, code after is unreachable (ignoring `else`).
   - `else` runs only if `try` did NOT raise, so it cannot make the overall block terminate.
   - `finally` always runs, but doesn't affect reachability of code *after* the `try` block — it affects reachability of code *within* `finally` itself.

2. **Handle bare `except:` vs typed `except ExcType:`**: With typed `except` clauses, there may be exceptions not covered by any handler. Only a bare `except:` (or `except BaseException:`) guarantees all exceptions are caught. Without full coverage, the `try` block is not exhaustive, and code after it might be reached via an uncaught exception.

### Complexity impact

High. The clause-tracking logic is more complex than `if`/`else` because of the four-clause structure and the asymmetric semantics of `else` and `finally`. False positive risk is significant due to the exception coverage question.

---

## 4. Coconut-Specific Constructs

### 4a. `match`/`case` Blocks

Coconut's `match` statement compiles into `if`/`elif`/`else` chains. An exhaustive match (one that covers all possible values) makes code after it unreachable:

```coconut
def f(x):
    match x:
        case int():
            return 1
        case _:
            return 2
    print("unreachable")  # NOT detected
```

At the raw-lines level, this has been compiled into `if`/`elif`/`else`. If branch-exhaustive analysis (section 1) were implemented, this case would be covered automatically — provided the match compiled into a chain ending with `else` (which the wildcard `_` case does).

However, matches without a wildcard case:

```coconut
match x:
    case int():
        return 1
    case str():
        return 2
```

compile to `if`/`elif` *without* `else`. The branch analysis would correctly treat this as non-exhaustive, which is the right behavior: the match might not cover all types.

### 4b. Pipeline Expressions and Operator Chains

These are expression-level constructs and don't interact with unreachable code detection. A pipeline like `x |> f |> g` compiles to `(g)((f)(x))`, which is a single expression — not a control-flow statement. No future work needed here.

### 4c. `addpattern` Functions

`addpattern` compiles multiple function definitions into a single function with pattern-matching dispatch. Each pattern's body is processed separately by `proc_funcdef`. The current detection applies to each pattern body independently, which is correct. No future work needed.

---

## 5. Cross-Statement Terminator Sequences

### What is missed

```python
def f():
    x = 1
    return x
    y = 2       # detected (code after return)
    z = 3       # NOT separately reported
```

The current algorithm reports `y = 2` as unreachable (after `return`), then resets `last_terminator` to `None` and does not flag `z = 3`. This is by design — one diagnostic per terminator prevents a cascade of errors for every line in a long unreachable block.

### What could be improved

An alternative is to report the entire unreachable *range* — "lines N through M are unreachable after return on line K". This would require:

1. Continuing to scan after the first unreachable line is found
2. Tracking the start and end of the unreachable region
3. Reporting a single diagnostic with a line range

This is low-complexity but would change the error message format and the `strict_err_or_warn` API.

---

## 6. Module-Level Unreachable Code

### What is missed

The detection only runs inside `proc_funcdef`. Code at the module level is never checked:

```python
import sys
sys.exit(0)
print("unreachable")  # NOT detected (module level)
```

### What would be needed

A separate pass over the module's top-level statements, outside of `proc_funcdef`. This would require a different entry point — perhaps in `deferred_code_proc` after all function wrappers are resolved, or in a new post-processing step. The algorithm itself would be similar (track terminators at level 0 instead of level 1), but `sys.exit()` is a function call, not a keyword, making it fundamentally harder to detect with regex alone.

---

## 7. NOQA Support

### Current limitation

The `noqa_able=False` parameter means users cannot suppress the warning with `# NOQA`. This was a deliberate initial choice because at the raw-lines stage, the NOQA comment from the original source may not be reliably accessible — comments are partially processed during compilation.

### What would be needed

1. Check whether the Coconut-embedded comment on the *unreachable line* (or the *terminator line*) contains a NOQA marker
2. Verify that `split_comment` preserves NOQA annotations at this stage of compilation
3. If so, change `noqa_able=False` to `noqa_able=True`

This is low-complexity but requires testing with actual `.coco` files to verify comment preservation.

---

## 8. Precise Line Number Reporting

### Current limitation

When compiling with `-c` (string input), `extract_line_num_from_comment` returns `None` because no line-number comments are embedded. The error falls back to reporting the line of the `def` statement, not the terminator. When compiling `.coco` files, line numbers may be more accurate, but this has not been thoroughly tested.

### What would be needed

1. Ensure that `-c` input gets line-number comments embedded during compilation (this may require changes to the pre-processing pipeline)
2. Alternatively, compute the terminator's line offset from the `def` statement's `loc` and the terminator's position within `raw_lines`
3. Test with multi-function `.coco` files to verify accuracy

---

## Summary: Effort vs. Impact Matrix

| Enhancement | False negatives eliminated | Implementation complexity | False positive risk | Priority |
|---|---|---|---|---|
| Branch-exhaustive `if`/`else` | High — very common pattern | Medium-High | Medium (match blocks, ternary) | High |
| `while True` without `break` | Low — uncommon pattern | Low-Medium | Low | Low |
| `try`/`except` all-path termination | Medium — common in error handling | High | Medium (exception coverage) | Medium |
| `match`/`case` exhaustiveness | Covered by `if`/`else` analysis | None (free with #1) | N/A | N/A |
| Cross-statement range reporting | N/A (UX improvement) | Low | None | Low |
| Module-level detection | Low — rare pattern | Medium | Low | Low |
| NOQA support | N/A (UX improvement) | Low | None | Medium |
| Precise line numbers | N/A (UX improvement) | Medium | None | Medium |

---

## Recommended Implementation Order

1. **NOQA support** — low effort, high user-experience impact, no algorithm changes
2. **Precise line numbers** — medium effort, makes diagnostics actionable
3. **Branch-exhaustive `if`/`else`** — the single highest-impact improvement; subsumes `match`/`case` analysis for free
4. **`try`/`except` all-path termination** — structurally similar to #3 but with more edge cases
5. **Cross-statement range reporting** — polish; do after the detection scope is finalized
6. **`while True` without `break`** — niche; do if users request it
7. **Module-level detection** — requires a new entry point; do last

---

## Architectural Constraint: No AST

All of these improvements must work within Coconut's **no-AST, single-pass, line-by-line** architecture. Traditional compilers perform unreachable code analysis on a control-flow graph (CFG) derived from an AST. Coconut has neither. Every enhancement described above must be implemented as additional state tracking in the existing line-scanning loop.

This is the fundamental reason the current design is conservative: a line-by-line scanner with flat state can trivially detect "terminator followed by code at the same level," but anything requiring cross-scope reasoning (branches, exception handlers) demands a state stack that simulates the structural information an AST would provide for free.

The practical ceiling for this approach is roughly the `if`/`else` + `try`/`except` analysis described above. Beyond that — e.g., detecting that a variable assignment makes a later condition always-true, rendering an `else` branch unreachable — would require symbolic execution or data-flow analysis, which is firmly outside the scope of a line-scanning post-processor.
