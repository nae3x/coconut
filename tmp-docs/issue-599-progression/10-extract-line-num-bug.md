# Phase 10: The `extract_line_num_from_comment` Bug — Detection Never Fires

## Summary

The unreachable code detector (`detect_unreachable_code`) never actually fires. It silently skips every line because `extract_line_num_from_comment` cannot parse line numbers at the compilation stage where the detector runs. The function expects `# N` format comments, but at the `deferred_code_proc` stage, line numbers are still in internal `lnwrapper` format (`‡N⏹`). Every call returns `None`, causing the detector to either skip the warning (thinking the line is compiler-generated) or record a `None` line number for the terminator.

This went undetected because `make test-univ` only tests the absence of false positives (compiling `.coco` files with `--strict`). The `main_test.py` positive tests (which assert that unreachable code *is* detected) were never run as part of `make test-univ` — they require `pytest` — and they fail when run.

---

## Two Line Number Formats in the Compiler

The Coconut compiler uses two different representations for source line numbers at different stages of compilation:

### Format 1: `lnwrapper` markers (internal, pre-`endline_repl`)

During parsing, line numbers are embedded as Unicode marker sequences:

```
‡N⏹
```

Where `‡` is `lnwrapper` (U+2021, double dagger) and `⏹` is `unwrapper` (U+23F9, stop square). For example, `return 1` on source line 2 becomes:

```
return 1‡2⏹
```

These markers are defined in `coconut/constants.py`:

```python
lnwrapper = "\u2021"  # double dagger
unwrapper = "\u23f9"  # stop square
```

The `lnwrapper` character is listed in `comment_chars` alongside `#`:

```python
comment_chars = ("#", lnwrapper)
```

This means `split_comment()` treats `lnwrapper` as a comment delimiter — it splits `return 1‡2⏹` into `base="return 1"` and `comment="‡2⏹"`. This part works correctly.

### Format 2: `# N` comments (final, post-`endline_repl`)

The `endline_repl` method (compiler.py:2398) converts `lnwrapper` markers to human-readable `# N` comments in the final output:

```python
def endline_repl(self, inputstring, ...):
    for line in literal_lines(inputstring):
        lnwrapper_split = line.split(lnwrapper)
        has_wrapped_ln = len(lnwrapper_split) > 1
        if has_wrapped_ln:
            line = lnwrapper_split.pop(0).rstrip()
            for ln_str in lnwrapper_split:
                new_ln = int(ln_str[:-1])  # strip unwrapper, parse int
                ln = new_ln
            line += self.ln_comment(src_ln)  # appends "# 2 (line in Coconut source)"
```

After `endline_repl`, the line becomes:

```
return 1 # 2 (line in Coconut source)
```

---

## The Pipeline Order

The post-processing pipeline runs these steps in sequence:

```
deferred_code_proc()     ← detect_unreachable_code runs HERE (lnwrapper format)
    └─ proc_funcdef()
        └─ detect_unreachable_code()
passthrough_repl()
reind_proc()
endline_repl()           ← converts lnwrapper → "# N" format
str_repl()
header_proc()
polish()
```

`detect_unreachable_code` runs during `deferred_code_proc`, **before** `endline_repl`. At this stage, line numbers are in `lnwrapper` format (`‡2⏹`), not `# N` format.

---

## What `extract_line_num_from_comment` Expects

The function (compiler/util.py:1925) is designed for `# N` format:

```python
def extract_line_num_from_comment(line, default=None):
    """Extract the line number from a line with a line number comment, else return default."""
    _, all_comments = split_comment(line)
    for comment in all_comments.split("#"):       # ← splits on "#"
        words = comment.strip().split(None, 1)
        if words:
            first_word = words[0].strip(":")
            try:
                return int(first_word)             # ← tries to parse integer
            except ValueError:
                pass
    return default
```

It calls `split_comment(line)` to extract the comment portion, then splits that on `"#"` to isolate individual comments, and tries to parse the first word as an integer.

### What happens with `# 2 (line in Coconut source)` (post-`endline_repl`)

```
all_comments = "# 2 (line in Coconut source)"
split on "#" → ["", " 2 (line in Coconut source)"]
words = ["2", "(line in Coconut source)"]
first_word = "2"
int("2") → 2 ✓
```

This works. The function was designed for this format.

### What happens with `‡2⏹` (pre-`endline_repl`, where the detector runs)

```
all_comments = "‡2⏹"
split on "#" → ["‡2⏹"]          ← no "#" found, returns the whole string
words = ["‡2⏹"]
first_word = "‡2⏹"
int("‡2⏹") → ValueError         ← the Unicode markers aren't digits
```

Returns `None`. **The line number is present but unparseable.**

---

## How This Breaks the Detector

The detector uses `extract_line_num_from_comment` in two places:

### Place 1: The unreachable code check (line 2654)

```python
if last_terminator is not None:
    # skip compiler-generated code (e.g. `if False: yield` from `yield def`)
    # which has no source line number comment
    raw_ln = extract_line_num_from_comment(comment)    # ← always None
    if raw_ln is not None:                              # ← always False
        # ... emit warning ...
    last_terminator = None
```

This was the Phase 8 (v2) fix: skip compiler-generated lines that have no source line number. But because `extract_line_num_from_comment` returns `None` for the `lnwrapper` format, **every line** looks like compiler-generated code. The warning is never emitted.

### Place 2: Recording the terminator line number (line 2669)

```python
m = self.terminator_stmt_regex.match(base)
if m:
    raw_ln = extract_line_num_from_comment(comment)    # ← always None
    term_ln = self.adjust(raw_ln) if raw_ln is not None else None  # ← always None
    last_terminator = (m.group(1), term_ln)
```

The terminator is recorded with `term_ln=None`. Even if the warning were emitted, the error message would have no line number.

### The combined effect

1. Terminator is found → `last_terminator = ("return", None)` ✓ (works despite None line number)
2. Next line at level 1 → checks `last_terminator is not None` → True ✓
3. Checks `extract_line_num_from_comment(comment)` → `None` ✗
4. `if raw_ln is not None` → False → **warning skipped**
5. `last_terminator = None` → state reset

The detection loop runs correctly, but the warning is silently swallowed at step 4 because the Phase 8 guard (`if raw_ln is not None`) treats every line as compiler-generated.

---

## Concrete Trace

Input: `def f():\n    return 1\n    x = 2\n`

After parsing, the raw_lines in `proc_funcdef` are:

```
line 0: 'return 1‡2⏹\n'
line 1: 'x = 2‡3⏹\n'
line 2: '‡4⏹\n'
```

The detector processes:

```
Line 0: base='return 1'  comment='‡2⏹'
  level=1, not suppressed, not blank
  last_terminator is None → skip warning check
  terminator_stmt_regex matches 'return'
  extract_line_num_from_comment('‡2⏹') → None
  last_terminator = ('return', None)

Line 1: base='x = 2'  comment='‡3⏹'
  level=1, not suppressed, not blank
  last_terminator is ('return', None) → CHECK FOR UNREACHABLE CODE
  extract_line_num_from_comment('‡3⏹') → None    ← THE BUG
  raw_ln is None → skip warning                   ← SILENTLY SWALLOWED
  last_terminator = None

Line 2: base=''  comment='‡4⏹'
  blank → skipped
```

The unreachable `x = 2` is detected but the warning is never emitted.

---

## Why `make test-univ` Passes

`make test-univ` runs:

```bash
python ./coconut/tests --strict --keep-lines --force
python ./coconut/tests/dest/runner.py
python ./coconut/tests/dest/extras.py
```

This compiles all `.coco` test files with `--strict` and runs them. It tests:

1. **No false positives** — the existing test suite compiles without the detector erroneously flagging any code
2. **Runtime correctness** — the compiled Python runs and all assertions pass

It does **not** test that the detector actually detects unreachable code. The negative tests (no false positives) pass vacuously — the detector never fires at all, so it cannot produce false positives.

The positive tests in `main_test.py` (which assert that unreachable code *is* detected) are only run via `pytest`:

```bash
pytest coconut/tests/main_test.py -k unreachable -v
```

These tests fail:

```
FAILED test_strict_unreachable_code_error - assert 0 == 1
FAILED test_strict_unreachable_code_warning - Expected 'found unreachable code after return statement'
```

The error test expects `retcode=1` but gets `0` (no error raised). The warning test expects the warning message in output but it's absent.

---

## Why `endline_repl` Has No Problem

`endline_repl` (compiler.py:2407) parses `lnwrapper` format correctly because it was specifically written for it:

```python
lnwrapper_split = line.split(lnwrapper)    # splits on ‡
for ln_str in lnwrapper_split:
    new_ln = int(ln_str[:-1])              # strips ⏹, parses "2" → 2
```

It splits on `lnwrapper` (not `#`), strips the `unwrapper`, and parses the remaining digits. This is the canonical parser for the `lnwrapper` format.

`extract_line_num_from_comment` was written for a different context — code that has already been through `endline_repl`, where line numbers appear as `# N`. It is used elsewhere in the codebase where the `# N` format is available, and works correctly there.

---

## Root Cause

The bug is a format mismatch: `detect_unreachable_code` runs at a pipeline stage where line numbers are in `lnwrapper` format, but calls `extract_line_num_from_comment` which only understands `# N` format.

This mismatch was introduced in the Phase 8 (v2) fix (commit `1e3bd9ba`), which added the `extract_line_num_from_comment` guard to distinguish user code from compiler-generated code. The guard condition (`if raw_ln is not None`) was intended to skip only compiler-generated lines (which truly have no line number), but it skips *all* lines because the function cannot parse the line number format present at this stage.

The original Phase 3 implementation (commit `b259d43c`) did not have this guard — it would have fired unconditionally. The bug was introduced specifically by the compiler-generated code skip logic.

---

## How to Fix

The detector needs to parse `lnwrapper` format directly instead of using `extract_line_num_from_comment`. The parsing logic from `endline_repl` can be adapted:

```python
def extract_ln_from_lnwrapper(comment):
    """Extract line number from lnwrapper-format comment (‡N⏹)."""
    parts = comment.split(lnwrapper)
    for part in parts[1:]:  # skip everything before the first lnwrapper
        if part.endswith(unwrapper):
            try:
                return int(part[:-1])
            except ValueError:
                pass
    return None
```

This replaces both calls to `extract_line_num_from_comment` in `detect_unreachable_code` (lines 2654 and 2669).

With this fix:
- User code lines have `‡N⏹` → parsed to line number N → warning is emitted
- Compiler-generated lines (like `if False: yield` from `yield def`) have no `lnwrapper` markers → returns `None` → warning is correctly skipped
