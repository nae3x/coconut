# Question for the repository owner regarding issue #599

## Draft question to post on the issue / PR

---

**Title:** Seeking guidance on unreachable code detection approach — interaction with `yield def` and line number formats

Hi @evhub,

I've been working on #599 (detect unreachable code after `return`/`raise` in `--strict` mode). The core detection works — a single-pass scanner in `proc_funcdef` that flags code after unconditional terminators at the function body's top level. However, I've run into a chain of issues around `yield def` and would like your guidance before going further.

### What works

The detection algorithm itself is sound: it iterates `raw_lines` in `proc_funcdef` (same pattern as `detect_is_gen` and `transform_returns`), tracks nesting via `openindent`/`closeindent`, and flags unreachable code at level 1. Tests cover all suppression boundaries (nested `def`, `for`, `while`, `try`, `with`, `if`). `make test-univ` passes with no false positives.

### The `yield def` problem

`yield def f(x) = x` compiles the body to `return x`, then `keyword_funcdef_handle` **appends** `if False: yield` at the bottom. The detector sees code after `return` and flags it — a false positive since the code is compiler-generated.

### Fix attempts and cascading issues

1. **Prepend instead of append** (move `if False: yield` to the top of the body) — this broke PEP 257 docstrings (`__doc__` not set because `if False: yield` becomes the first statement).

2. **Keep append, skip lines without source line numbers** (compiler-generated code has no `lnwrapper` markers) — this is the current approach. The logic is correct in principle, but I used `extract_line_num_from_comment()` to check for line numbers, which only parses `# N` format. At the `deferred_code_proc` stage, line numbers are still in `lnwrapper` format (`‡N⏹`), so the function always returns `None` and the detector silently skips **all** lines — it never fires.

### Current state

The detection code is in place and the algorithm is correct, but the line number parsing mismatch means warnings are never emitted. Fixing this requires either:

- **(A)** Adding a new utility `extract_line_num_from_lnwrapper` that parses the `‡N⏹` format (mirroring `endline_repl`'s logic), and using it in `detect_unreachable_code` instead of `extract_line_num_from_comment`.

- **(B)** A different approach to the `yield def` interaction — e.g., having `keyword_funcdef_handle` mark the inserted code so the detector can recognize it without relying on line number presence/absence.

- **(C)** Your preferred alternative — I may be overcomplicating this.

### Questions

1. Is approach **(A)** (new lnwrapper parser) acceptable, or would you prefer a different mechanism to distinguish compiler-generated code from user code?

2. For `keyword_funcdef_handle`, is the append-at-bottom behavior for `if False: yield` something you'd want preserved, or would you be open to a different insertion strategy (e.g., a dedicated marker that the detector can check)?

3. Any other concerns about the detection running in `proc_funcdef` / `deferred_code_proc`?

---

### Branch: `fix-599-yield-error`

### Relevant commits:
```
b259d43c Apply fix for issue 599
ab2c5e71 Fix case detect unreachable code when yield
1e3bd9ba Revert yield insertion to bottom; skip unreachable warning for compiler-generated code
9610d3fb Test addition (main_test.py) — the yield def regression test
```
