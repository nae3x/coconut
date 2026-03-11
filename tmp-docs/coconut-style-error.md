# CoconutStyleError — Detailed Explanation

## What Is It?

`CoconutStyleError` is an exception class defined in `coconut/exceptions.py:249`. It represents **code style violations** — problems that are not syntactically invalid but indicate poor practice or likely bugs. It is the mechanism Coconut uses to enforce stricter code quality when the `--strict` flag is enabled.

```
CoconutStyleError: found unused import 'abc' (add '# NOQA' to suppress) (remove --strict to downgrade to a warning) (line 1)
```

The key behavior is **dual-mode**: the same style check either **fails compilation** (with `--strict`) or **prints a warning** (without `--strict`).

---

## Class Hierarchy

```
BaseException
└── BaseCoconutException
    └── Exception
        └── CoconutException
            ├── CoconutSyntaxError          # base for all syntax-related errors
            │   ├── CoconutStyleError       # --strict mode style violations
            │   ├── CoconutTargetError      # --target version errors
            │   ├── CoconutParseError       # parsing failures
            │   └── CoconutSyntaxWarning    # warnings with syntax error formatting
            └── CoconutWarning              # base for non-fatal warnings
```

`CoconutStyleError` inherits from `CoconutSyntaxError`, which gives it:
- Rich error formatting with source code display, line numbers, and squiggly underlines
- A `.syntax_err()` method to convert to a Python `SyntaxError`
- Support for `point`, `endpoint`, `ln`, `filename`, and `extra` parameters

---

## Constructor

```python
class CoconutStyleError(CoconutSyntaxError):
    """Coconut --strict error."""

    def __init__(self, message, source=None, point=None, ln=None,
                 extra="remove --strict to dismiss", endpoint=None, filename=None):
        self.args = (message, source, point, ln, extra, endpoint, filename)
```

| Parameter  | Type       | Description |
|------------|-----------|-------------|
| `message`  | `str`     | The error description (e.g., `"found unused import 'abc'"`) |
| `source`   | `str`     | The original source code string for context display |
| `point`    | `int`     | Character offset in `source` where the error starts |
| `ln`       | `int`     | Line number (after adjustment) for display in the error message |
| `extra`    | `str`     | Hint text shown in parentheses (defaults to `"remove --strict to dismiss"`) |
| `endpoint` | `int`     | Character offset where the error region ends (for squiggly underlines) |
| `filename` | `str`     | Source file name for display |

The default `extra` value is `"remove --strict to dismiss"`, but when raised through `strict_err_or_warn`, it is overridden to `"remove --strict to downgrade to a warning"`.

---

## How To Use It: The Three APIs

You should **never** raise `CoconutStyleError` directly. The `Compiler` class provides three helper methods that manage the dual-mode behavior automatically.

### 1. `strict_err_or_warn` — Primary API (most common)

**File**: `coconut/compiler/compiler.py:1159`

```python
def strict_err_or_warn(self, msg, original, loc, noqa_able=False, **kwargs):
```

This is the method used by the vast majority of style checks. It handles two modes:

- **`self.strict == True`**: Raises a `CoconutStyleError` → compilation **fails**
- **`self.strict == False`**: Calls `self.syntax_warning()` → prints a `CoconutSyntaxWarning` and compilation **continues**

**Parameters**:
- `msg`: Error message string
- `original`: The full source being compiled
- `loc`: Character offset where the issue was detected
- `noqa_able` (default `False`): If `True`, checks for `# NOQA` comments and adds `"(add '# NOQA' to suppress)"` to the message
- `**kwargs`: Passed through to `make_err` — commonly `ln=`, `endpoint=`, `raise_err_func=`

**Internal behavior**:

```python
def strict_err_or_warn(self, msg, original, loc, noqa_able=False, **kwargs):
    raise_err_func = kwargs.pop("raise_err_func", None)
    pure_err = kwargs.pop("pure_err", False)

    if noqa_able:
        if self.has_noqa_comment(original, loc):
            return None                          # suppressed by # NOQA
        msg += " (add '# NOQA' to suppress)"

    if self.strict:
        kwargs["extra"] = "remove --strict to downgrade to a warning"
        if raise_err_func is None:
            raise self.make_err(CoconutStyleError, msg, original, loc, **kwargs)
        else:
            return raise_err_func(CoconutStyleError, msg, original, loc, **kwargs)
    else:
        self.syntax_warning(msg, original, loc, **kwargs)
```

### 2. `strict_err` — Error-Only (no warning fallback)

**File**: `coconut/compiler/compiler.py:1133`

```python
def strict_err(self, *args, **kwargs):
    """Raise a CoconutStyleError if in strict mode."""
    if self.strict:
        raise self.make_err(CoconutStyleError, *args, **kwargs)
```

Silently does nothing when `--strict` is off. Used rarely — only when a warning would be noise.

### 3. `pure_error` — Strict + Pure Mode

**File**: `coconut/compiler/compiler.py:1185`

```python
def pure_error(self, msg, original, loc, **kwargs):
    if self.pure:
        return self.strict_err_or_warn(msg, original, loc, pure_err=True, **kwargs)
```

Only triggers when `--pure` is enabled. The `extra` message includes `"remove --pure to dismiss"` in addition to the strict hint.

---

## How `make_err` Constructs the Error

All three APIs delegate to `make_err` (`compiler.py:1371`):

```python
def make_err(self, errtype, message, original, loc=0, ln=None, extra=None,
             reformat=True, endpoint=None, ...):
```

This method:
1. Adjusts `loc` to point at non-whitespace (so the error indicator is useful)
2. Computes `endpoint` for squiggly underline ranges
3. Adjusts line numbers via `self.adjust(ln)` (maps internal line numbers to source line numbers)
4. Constructs and returns `errtype(message, source, point, ln, extra, endpoint, filename)`

---

## What Style Checks Use `CoconutStyleError`?

Here is a categorized list of all style checks in the codebase, showing the message and where they are triggered:

### Code Quality Checks
| Message | Location | `noqa_able` |
|---------|----------|-------------|
| `"found unused import '{name}'"` | `compiler.py:1558` | Yes |
| `"found undefined name '{name}'"` | `compiler.py:5904` | Yes |
| `"found unreachable code after {kwd} statement"` | `compiler.py:2652` | No |
| `"found implicit string concatenation"` | `compiler.py:5274` | Yes |
| `"found mixing of tabs and spaces"` | `compiler.py:2226` | No |

### Deprecation Warnings
| Message | Location | `noqa_able` |
|---------|----------|-------------|
| `"'...={name}' shorthand is deprecated"` | `compiler.py:3325` | Yes |
| `"'obj.' as a shorthand for 'getattr$(obj)' is deprecated"` | `compiler.py:3546` | Yes |
| `"deprecated case keyword at top level"` | `compiler.py:4916` | Yes |
| `"use of '<=' as a type parameter bound declaration operator is deprecated"` | `compiler.py:5430` | No |

### Redundancy Checks
| Message | Location | `noqa_able` |
|---------|----------|-------------|
| `"unnecessary inheriting from object"` | `compiler.py:3765` | Yes |
| `"unnecessary from __future__ import"` | `compiler.py:4294` | Yes |
| `"f-string with no expressions"` | `compiler.py:4965` | Yes |

### Shadowing Checks
| Message | Location | `noqa_able` |
|---------|----------|-------------|
| `"assignment shadows builtin '{name}'"` | `compiler.py:5811` | No |
| `"assignment shadows imported '{name}'"` | `compiler.py:5823` | No |

### Style Enforcement
| Message | Location | `noqa_able` |
|---------|----------|-------------|
| `"found statement lambda followed by comma"` | `compiler.py:4603` | No |

---

## `noqa_able` — NOQA Comment Suppression

When `noqa_able=True`:
1. The method checks if the line at `loc` has a `# NOQA` comment
2. If found, the check is silently suppressed (returns `None`)
3. If not found, `"(add '# NOQA' to suppress)"` is appended to the message

When `noqa_able=False`:
- No NOQA check is performed — the error/warning always fires
- This is used when the check runs at a stage where comments haven't been parsed yet, or when suppression doesn't make sense

---

## Error Message Format

A full `CoconutStyleError` message looks like:

```
found unused import 'abc' (add '# NOQA' to suppress) (remove --strict to downgrade to a warning) (line 1)
    import abc
    ^~~~~~~~~~|
```

The components:
1. **Message**: `"found unused import 'abc'"`
2. **NOQA hint** (if `noqa_able`): `"(add '# NOQA' to suppress)"`
3. **Extra** (strict hint): `"(remove --strict to downgrade to a warning)"`
4. **Line number**: `"(line 1)"`
5. **Source context + squiggles**: visual pointer to the problematic code

In warning mode (no `--strict`), the same message is printed as a `CoconutSyntaxWarning` via `logger.warn_err()`, and compilation continues.

---

## How It Relates to Issue 599

Issue 599 requests detection of **unreachable code** — statements after an unconditional terminator (`return`, `raise`, `break`, `continue`) at the top level of a function body.

The implementation (`compiler.py:2652`) uses `strict_err_or_warn` exactly like other style checks:

```python
self.strict_err_or_warn(
    "found unreachable code after " + term_kwd + " statement",
    original,
    loc,
    ln=term_ln,
    endpoint=False,
)
```

This means:
- **With `--strict`**: `CoconutStyleError` is raised → compilation **fails**
  ```
  CoconutStyleError: found unreachable code after return statement
    (remove --strict to downgrade to a warning) (line 3)
  ```
- **Without `--strict`**: `CoconutSyntaxWarning` is printed → compilation **succeeds**
  ```
  CoconutWarning: found unreachable code after return statement (line 3)
  ```

`noqa_able` is set to `False` because the check runs during post-processing (`proc_funcdef`), at which point comments have not been parsed for the function body lines. `endpoint` is set to `False` (which sets `endpoint = loc` internally), meaning no squiggly underline range is shown.

The unreachable code detection is architecturally identical to existing checks like unused imports and undefined names — it's just another style check plugged into the same `strict_err_or_warn` mechanism.

---

## Testing `CoconutStyleError`

In the test suite (`coconut/tests/src/extras.coco`), `CoconutStyleError` is tested using `assert_raises`:

```python
# expect a CoconutStyleError with a specific substring
assert_raises(-> parse_with_state("lambda x: x"), CoconutStyleError)
assert_raises(-> parse_with_state("abc "), CoconutStyleError, err_has="\n     ^")
```

For the unreachable code feature (issue 599), tests are in `coconut/tests/main_test.py` using `call_coconut`:

```python
def test_strict_unreachable_code_error(self):
    call_coconut(
        ["--strict", "-c", "def f():\n    return 1\n    x = 2\n"],
        expect_retcode=1,
        check_errors=False,
        assert_output="found unreachable code after return statement",
    )
```
