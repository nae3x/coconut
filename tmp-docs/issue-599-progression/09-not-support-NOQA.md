# Issue 599: Why the Unreachable Code Checker Cannot Support `# NOQA` Suppression

## Background

Coconut's `--strict` mode already has infrastructure for `# NOQA` comment suppression. Many strict checks (unused imports, deprecated shorthands, unnecessary `object` inheritance, etc.) allow users to add `# NOQA` on the offending line to silence the warning/error. The unreachable code checker introduced for issue 599 does **not** support this. This document explains why.

---

## How the Existing `# NOQA` Mechanism Works

The suppression system has three components:

### 1. Comment Stripping (`comment_handle`, compiler.py:3689)

During parsing, Coconut strips user comments from the source and stores them in a dictionary keyed by **adjusted line number**:

```python
def comment_handle(self, original, loc, tokens):
    comment_marker, = tokens
    ln = self.adjust(lineno(loc, original))
    self.comments[ln].add(comment_marker)
    return ""  # comment is removed from the compiled output
```

After this step, the compiled code contains no user comments — only compiler-generated line-number markers (e.g., `‡23⏹`). The original comments are preserved in `self.comments`, a `defaultdict(set)` mapping `adjusted_line_number → {comment_marker, ...}`.

### 2. NOQA Lookup (`has_noqa_comment`, compiler.py:1153)

To check if a specific source location has a `# NOQA` comment:

```python
def has_noqa_comment(self, original, loc):
    ln = self.adjust(lineno(loc, original))
    comment = self.reformat(" ".join(self.comments[ln]), ignore_errors=True)
    return self.noqa_regex.search(comment)
```

This method:
1. Converts the **parse location** `loc` (a character offset into `original`) to a **line number** using PyParsing's `lineno(loc, original)`.
2. Adjusts the line number for any preprocessing transformations via `self.adjust()`.
3. Looks up the user's comments for that line in `self.comments[ln]`.
4. Searches for the case-insensitive pattern `noqa` in those comments.

### 3. Gated Check in `strict_err_or_warn` (compiler.py:1159)

```python
def strict_err_or_warn(self, msg, original, loc, noqa_able=False, **kwargs):
    if noqa_able:
        if self.has_noqa_comment(original, loc):
            return None  # suppressed — do nothing
        msg += " (add '# NOQA' to suppress)"
    # ... raise error or warning ...
```

When `noqa_able=True`, the method calls `has_noqa_comment(original, loc)` before raising the error. If a NOQA comment is found on the line identified by `loc`, the error is silently suppressed.

---

## How the Unreachable Code Checker Calls the Error

The checker is invoked from `proc_funcdef` (compiler.py:2926):

```python
# inside proc_funcdef(self, original, loc, decorators, funcdef, ...):
self.detect_unreachable_code(original, loc, raw_lines)
```

Where:
- **`original`**: the entire source file as a string (the full input to the parser).
- **`loc`**: the **parse location of the function definition** — i.e., the character offset in `original` where the `def` keyword was matched by PyParsing.
- **`raw_lines`**: the compiled body lines of the function (with indent markers and line-number comments, but **without** user comments).

When the checker finds unreachable code (compiler.py:2651–2660):

```python
if last_terminator is not None:
    term_kwd, term_ln = last_terminator
    self.strict_err_or_warn(
        "found unreachable code after " + term_kwd + " statement",
        original,
        loc,          # ← the function definition's parse location
        ln=term_ln,
        noqa_able=False,
        endpoint=False,
    )
```

It passes `original` and `loc` straight through from `proc_funcdef`.

---

## The Core Problem: `loc` Points to the Wrong Line

This is the fundamental reason `noqa_able=True` cannot simply be enabled.

### What `loc` actually is

`loc` is the character offset where PyParsing matched the function definition. For a function like:

```coconut
def it_ret(x):        # line 23
    return x          # line 24
    yield None        # line 25  ← user wants # NOQA here
```

`loc` points to the `def` keyword on **line 23**. PyParsing's `lineno(loc, original)` would return **23**.

### What `has_noqa_comment` would check

If we set `noqa_able=True`, the call chain would be:

1. `strict_err_or_warn` calls `has_noqa_comment(original, loc)`
2. `has_noqa_comment` computes `ln = self.adjust(lineno(loc, original))` → **line 23**
3. It looks up `self.comments[23]` — the comments on the `def it_ret(x):` line
4. It searches for `noqa` in those comments

### Where the user would put `# NOQA`

The user would naturally write:

```coconut
def it_ret(x):
    return x
    yield None  # NOQA
```

The `# NOQA` comment is on **line 25**. It is stored in `self.comments[25]`. But `has_noqa_comment` looks at `self.comments[23]` (the function definition line). **The comment is never found.**

### Summary of the mismatch

| Entity | Line | What it contains |
|--------|------|------------------|
| `loc` (from `proc_funcdef`) | 23 (`def it_ret(x):`) | Parse location of the function definition |
| `has_noqa_comment` checks | 23 | Comments on the `def` line |
| Terminator (`return x`) | 24 | The statement that makes subsequent code unreachable |
| Unreachable code (`yield None`) | 25 | The line the user wants to suppress |
| User's `# NOQA` comment | 25 | Where the user would naturally place the suppression |

The checker has `term_ln` (line 24, the terminator line) and could extract the unreachable line number from the current line's comment marker. But neither of these is `loc`. And `has_noqa_comment` only accepts `(original, loc)` — it has no parameter for an explicit line number.

---

## Why Other `noqa_able=True` Checks Don't Have This Problem

Other checks that use `noqa_able=True` work correctly because their `loc` points to the **exact token** that triggered the warning. For example:

### Unused imports (compiler.py:1558)

```python
for loc in info["imported"]:
    self.strict_err_or_warn("found unused import ...", original, loc, noqa_able=True, ...)
```

Here, `loc` is the parse location where the specific `import` statement was matched. `lineno(loc, original)` returns the line of the import. If the user writes `import foo  # NOQA`, the comment is on the same line that `loc` points to. The lookup succeeds.

### Deprecated shorthand (compiler.py:3327)

```python
self.strict_err_or_warn("'...={name}' shorthand is deprecated ...", original, loc, noqa_able=True)
```

Here, `loc` is the parse location of the specific shorthand expression. Same principle: `loc` points to the exact line being warned about.

### The pattern

All existing `noqa_able=True` checks are triggered directly by a **parse action** (a handler attached to a grammar rule). The PyParsing framework invokes the handler with `(original, loc, tokens)`, where `loc` is the character offset of the matched token. So `loc` always corresponds to the correct source line.

The unreachable code checker is **not** a parse action. It is a post-processing analysis invoked from `proc_funcdef`, which receives `loc` for the function definition as a whole. By the time the checker identifies which specific line is unreachable, it has no `loc` for that line — only compiled body lines with embedded line-number comments.

---

## What Would Be Needed to Fix This

To support `# NOQA` in the unreachable code checker, the code would need to bypass `has_noqa_comment` and directly look up `self.comments` using the line number extracted from the compiled code. Conceptually:

```python
# Inside detect_unreachable_code, when unreachable code is found:
if last_terminator is not None:
    term_kwd, term_ln = last_terminator

    # Extract the line number of the unreachable code from its comment marker
    unreachable_raw_ln = extract_line_num_from_comment(comment)
    unreachable_ln = self.adjust(unreachable_raw_ln) if unreachable_raw_ln is not None else None

    # Check NOQA on the terminator line and/or the unreachable line
    should_suppress = False
    for check_ln in (term_ln, unreachable_ln):
        if check_ln is not None and check_ln in self.comments:
            noqa_comment = self.reformat(
                " ".join(self.comments[check_ln]), ignore_errors=True
            )
            if self.noqa_regex.search(noqa_comment):
                should_suppress = True
                break

    if not should_suppress:
        self.strict_err_or_warn(
            "found unreachable code after " + term_kwd + " statement",
            original,
            loc,
            ln=term_ln,
            noqa_able=False,  # still False — we handle NOQA ourselves above
            endpoint=False,
        )
    last_terminator = None
```

This skips the `has_noqa_comment(original, loc)` path entirely and instead looks up comments by the actual line numbers of the terminator and unreachable code. The user could then place `# NOQA` on either the `return` line or the unreachable line to suppress the error.
