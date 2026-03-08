# Plan: Add Comprehensive Tests for Unreachable Code Detection

## Context

The unreachable code detection feature (issue #599) currently has only two tests: one verifying `--strict` mode produces an error, and one verifying non-strict mode produces a warning. Both test the same scenario (`return` + code after). This leaves the detection's behavioral boundaries undocumented through tests. A contributor reading the tests would not understand what the detector intentionally skips (nested functions, loops, try blocks, if-branches) or why.

The goal is to add tests that serve as **executable documentation** of the detection scope, covering all terminator types and all suppression mechanisms.

## File to Modify

`coconut/tests/main_test.py` -- insert new test methods at line 1118, between the existing `test_strict_unreachable_code_warning` (line 1110) and the `if get_bool_env_var("COCONUT_TEST_VERBOSE"):` block (line 1119).

## Tests to Add (11 new methods)

### Positive Tests (should detect unreachable code)

| # | Method Name | Code String | Asserts | Rationale |
|---|---|---|---|---|
| 1 | `test_strict_unreachable_code_raise` | `def f():\n    raise ValueError\n    x = 2\n` | error with `"after raise statement"` | Tests `raise` as a terminator (different keyword than `return`) |
| 2 | `test_strict_unreachable_code_raise_with_arg` | `def f():\n    raise Exception('msg')\n    x = 2\n` | error with `"after raise statement"` | Tests `raise` with an argument still matches |

Both use: `["--strict", "-c", CODE], expect_retcode=1, check_errors=False, assert_output=..., assert_output_only_at_end=False`

**Why no `break`/`continue` positive tests**: These keywords at level 1 (function body top level) would be outside any loop, making them invalid Python syntax. The compiler would reject the code before reaching `detect_unreachable_code`.

### Negative Tests (should NOT detect unreachable code)

| # | Method Name | Code String | Rationale |
|---|---|---|---|
| 3 | `test_strict_unreachable_code_nested_def` | `def f():\n    def g():\n        return 1\n    x = 2\n` | `func_until_level` suppresses inner `def` |
| 4 | `test_strict_unreachable_code_for_loop` | `def f():\n    for i in range(10):\n        return i\n    x = 2\n` | `disabled_until_level` suppresses `for` |
| 5 | `test_strict_unreachable_code_while_loop` | `def f():\n    while True:\n        return 1\n    x = 2\n` | `disabled_until_level` suppresses `while` |
| 6 | `test_strict_unreachable_code_try_block` | `def f():\n    try:\n        return 1\n    except:\n        pass\n    x = 2\n` | `disabled_until_level` suppresses `try` |
| 7 | `test_strict_unreachable_code_with_block` | `def f(ctx):\n    with ctx:\n        return 1\n    x = 2\n` | `disabled_until_level` suppresses `with` |
| 8 | `test_strict_unreachable_code_if_no_else` | `def f():\n    if True:\n        return 1\n    x = 2\n` | `return` at level 2 inside `if` never sets `last_terminator` at level 1 |
| 9 | `test_strict_unreachable_code_if_else_all_return` | `def f():\n    if True:\n        return 1\n    else:\n        return 2\n    x = 3\n` | Conservative design: no branch analysis, so NOT detected |
| 10 | `test_strict_unreachable_code_return_only` | `def f():\n    return 1\n` | `last_terminator` set but no subsequent line at level 1 |
| 11 | `test_strict_unreachable_code_empty_function` | `def f():\n    pass\n` | `pass` is not a terminator; baseline sanity check |

All negative tests use: `call_coconut(["--strict", "-c", CODE])` with defaults (`expect_retcode=0`, `check_errors=True`). Using `--strict` ensures any false positive becomes a hard error that fails the test.

## Implementation Details

- All methods go inside `TestCompilation` class (4-space indent for method definition)
- Positive tests come first, then negative tests
- Each method has a descriptive docstring explaining the boundary being tested
- The `with` block test uses `def f(ctx):\n    with ctx:` instead of `with open('f') as fh:` to avoid referencing a nonexistent file (the code is compiled AND executed by `-c`)

## Verification

```bash
# Run only the unreachable code tests
pytest coconut/tests/main_test.py -k unreachable -v

# Expected: 13 tests pass (2 existing + 11 new)
```
