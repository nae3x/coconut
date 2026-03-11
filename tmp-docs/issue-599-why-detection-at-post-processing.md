# Why Unreachable Code Detection Must Happen at Post-Processing

This document explains why the unreachable code detection for issue #599 is implemented in the **post-processing phase** of the Coconut compiler, and why the other two phases (pre-processing and PyParsing) are not viable alternatives.

---

## Background: The Three-Phase Pipeline

The Coconut compiler transforms `.coco` source files into Python through three sequential phases:

```
Phase 1: PRE-PROCESSING    Phase 2: PYPARSING         Phase 3: POST-PROCESSING
(compiler.py - pre())      (grammar.py)               (compiler.py - post())

prepare()                  file_parser.parseString()   deferred_code_proc()  <-- HERE
str_proc()                                             passthrough_repl()
passthrough_proc()                                     reind_proc()
operator_proc()                                        endline_repl()
ind_proc()                                             str_repl()
                                                       header_proc()
                                                       polish()
```

Unreachable code detection runs inside `deferred_code_proc()`, the very first step of post-processing. Specifically, it is called from `proc_funcdef()`, which finalizes each deferred function definition.

---

## What Unreachable Code Detection Needs

To detect unreachable code, the compiler needs four things:

1. **Function body boundaries** -- know exactly where a function body starts and ends, so only the statements of *that* function are analyzed.
2. **Nesting-level tracking** -- distinguish between a `return` at the top level of the function body (which makes subsequent code unreachable) and a `return` inside a nested `if`, `for`, or inner `def` (which does not).
3. **Statement identification** -- determine which lines are control-flow terminators (`return`, `raise`, `break`, `continue`) and which are ordinary code.
4. **Source line numbers** -- map detected issues back to original Coconut source line numbers for error reporting.

Each phase is evaluated below against these four requirements.

---

## Why Not Pre-Processing?

Pre-processing (`pre()` in `compiler.py`) applies five mechanical text transformations in sequence:

```python
preprocs = [
    lambda self: self.prepare,          # normalize line endings
    lambda self: self.str_proc,         # hide string contents behind markers
    lambda self: self.passthrough_proc, # handle \ passthroughs
    lambda self: self.operator_proc,    # handle custom operators
    lambda self: self.ind_proc,         # convert indentation to openindent/closeindent
]
```

These transforms operate on raw text. They do not parse Coconut syntax.

### Problem 1: No function boundary knowledge

Pre-processing sees the entire file as a flat string. After `ind_proc()` runs, indentation is encoded as `openindent`/`closeindent` markers, so in principle you could search for a `def` keyword followed by an `openindent` and its matching `closeindent`. But Coconut function definitions are far more varied than plain `def`:

```
addpattern def f(x) = ...
copyclosure def f(x) = ...
async def f(x): ...
addpattern copyclosure async def f(x): ...
match def f(x, y): ...
```

The `def_regex` used in the actual detection already accounts for these variants:

```python
def_regex = compile_regex(r"((async|addpattern|copyclosure)\s+)*def\b")
```

But finding the *end* of the function body would require tracking matched `openindent`/`closeindent` pairs, skipping nested definitions, handling one-line `def f(x) = expr` forms, and dealing with all of Coconut's syntactic sugar. This amounts to writing a second, simplified Coconut parser -- duplicating work that PyParsing already does.

### Problem 2: Cannot distinguish nesting contexts

Even if you could identify function boundaries, you would need to know whether a `return` is at the function's direct body level or inside a nested structure. Consider:

```coconut
def f():
    if condition:
        return 1
    do_more()    # NOT unreachable -- the return is inside an if-branch
```

versus:

```coconut
def f():
    return 1
    do_more()    # UNREACHABLE -- the return is at the top level
```

After `ind_proc()`, both `if` blocks and nested `def` blocks produce `openindent`/`closeindent` markers. But there is no way to distinguish "this `openindent` starts an `if`-block" from "this `openindent` starts a nested `def`" without parsing the Coconut syntax that precedes it. The markers only encode *depth*, not *kind*.

In contrast, `proc_funcdef()` receives `raw_lines` that are already bounded to a single function body, with level 1 being the function's direct body. The algorithm uses `def_regex` and `tco_disable_regex` to classify each scope-opening line:

```python
# entering a nested def -- inner terminators don't affect outer scope
if func_until_level is None and self.def_regex.match(base):
    func_until_level = level

# entering a loop/try/with -- disable checking inside
if disabled_until_level is None and self.tco_disable_regex.match(base):
    disabled_until_level = level
```

Pre-processing has none of this context.

### Problem 3: Coconut-specific syntax is not yet resolved

Pre-processing does not understand Coconut constructs. For example, pattern-matching functions, pipeline expressions, and `case` blocks are still in Coconut syntax at this stage. A `return` might appear inside a Coconut `match` block that doesn't exist in Python. Without parsing, you cannot correctly interpret these constructs.

### Problem 4: String contents are hidden but not gone

After `str_proc()`, string literals are replaced with opaque markers like `\x00strwrapper0\x00`. A false `return` inside a string literal won't confuse the regex. However, passthrough blocks (`\\`-escaped Python) and operator definitions are only partially handled at this stage. The text is in an intermediate state that is not fully safe to analyze for control-flow semantics.

### Summary for pre-processing

| Requirement | Pre-processing capability |
|---|---|
| Function body boundaries | Cannot reliably identify (too many `def` variants, no end-of-body detection) |
| Nesting-level tracking | Has `openindent`/`closeindent` but cannot classify what they belong to |
| Statement identification | Regex could match `return`/`raise` but cannot distinguish contexts |
| Source line numbers | Available (not yet processed) but useless without correct detection |

**Verdict**: Pre-processing would require building a mini Coconut parser just for control-flow analysis, duplicating PyParsing's work.

---

## Why Not During PyParsing?

The PyParsing phase applies grammar rules with attached handlers that immediately emit Python code strings. The relevant architecture:

```python
# grammar.py: rule definition
normal_pipe_expr_tokens = OneOrMore(pipe_item) + last_pipe_item

# compiler.py: handler attachment
cls.normal_pipe_expr <<= attach(cls.normal_pipe_expr_tokens, cls.method("pipe_handle"))
```

Each handler returns a Python string. There is no AST. By the time PyParsing finishes, the input is consumed and replaced by one flat Python output string.

### Problem 1: No AST to walk

Coconut does not build an Abstract Syntax Tree. Traditional compilers (GCC, Clang, rustc, javac) perform unreachable code analysis by walking an AST or control-flow graph. In those compilers, the parser produces a tree structure, and a separate analysis pass examines it.

Coconut's parse actions *are* the compilation. When `pipe_handle` fires for `x |> f`, it immediately returns the string `(f)(x)`. The parsed tokens are consumed and discarded. There is no tree to walk after parsing, and there is no "function body node" to analyze.

### Problem 2: Function bodies are deferred

This is the critical architectural constraint. When PyParsing encounters a function definition, the handler (`decoratable_funcdef_stmt_handle`) does **not** process the function body. Instead, it wraps the entire function in a deferred marker:

```python
def decoratable_funcdef_stmt_handle(self, original, loc, tokens, is_async=False, is_stmt_lambda=False):
    """Wrap the given function for later processing."""
    if len(tokens) == 1:
        funcdef, = tokens
        decorators = ""
    elif len(tokens) == 2:
        decorators, funcdef = tokens
    # ...
    return funcwrapper + self.add_ref("func", (original, loc, decorators, funcdef, is_async, self.in_method, is_stmt_lambda)) + "\n"
```

The output is something like `_coconut_func_wrapper42\n` -- just a placeholder. The actual function code (including its body) is stored in a reference table (`self.add_ref("func", ...)`). It is **not available** as a complete, analyzable unit during the PyParsing phase.

Why is it deferred? Because Coconut needs to perform whole-body analyses (generator detection, tail-call optimization, return transformation) that require seeing the *entire* function body at once. PyParsing processes rules bottom-up and token-by-token; it cannot provide a "the entire function body is now complete" callback point.

### Problem 3: No natural "function complete" callback

Even if functions were not deferred, PyParsing grammar rules fire when they match. A `funcdef` rule would fire when the function definition is complete. But at that moment:

- The function body has already been transformed token-by-token into Python strings by inner parse actions.
- Each inner handler has already run independently and discarded its input.
- There is no way to "go back" and examine the original Coconut statements to check control flow.

You could theoretically add a parse action that collects all statements in a function body and analyzes them. But this would conflict with the deferred processing architecture that already exists for generator detection and tail-call optimization. It would mean duplicating or moving the entire `proc_funcdef` pipeline into a parse action, which breaks the established design.

### Summary for PyParsing

| Requirement | PyParsing capability |
|---|---|
| Function body boundaries | Function bodies are deferred -- not available as a unit |
| Nesting-level tracking | No AST; parse actions emit strings, not structural data |
| Statement identification | Could theoretically be added to a parse action, but body isn't available |
| Source line numbers | Available via `loc` parameter, but body analysis cannot happen here |

**Verdict**: Function bodies are architecturally unavailable during PyParsing. The deferred processing design makes this phase fundamentally unsuitable.

---

## Why Post-Processing Works

Post-processing begins with `deferred_code_proc()`, which iterates the partially-compiled output, finds deferred function markers, and calls `proc_funcdef()` to finalize each function:

```python
def deferred_code_proc(self, inputstring, ...):
    for raw_line in literal_lines(inputstring, True):
        bef_ind, line, aft_ind = split_leading_trailing_indent(raw_line)

        if line.startswith(funcwrapper):
            # Retrieve the deferred function data
            func_id = int(assert_remove_prefix(line, funcwrapper))
            original, loc, decorators, funcdef, is_async, in_method, is_stmt_lambda = self.get_ref("func", func_id)

            # Recursively process inner deferred code
            raw_funcdef = self.deferred_code_proc(tempsep + funcdef, ...)

            # Split into def line + body, then finalize
            func_code = ...
            out += [self.proc_funcdef(original, loc, decorators, func_code, ...)]
```

Inside `proc_funcdef()`, the function body is available as `raw_lines` -- a list of lines with clear boundaries:

```python
def proc_funcdef(self, original, loc, decorators, funcdef, is_async, in_method, is_stmt_lambda):
    raw_lines = list(literal_lines(funcdef, True))
    def_stmt = raw_lines.pop(0)  # Remove the def line itself

    # The three body-analysis steps, all operating on the same raw_lines:
    self.detect_unreachable_code(original, loc, raw_lines)   # <-- unreachable detection
    is_gen = self.detect_is_gen(raw_lines)                   # generator detection
    func_code, tco, tre = self.transform_returns(...)        # return transformation
```

### All four requirements are met

**1. Function body boundaries** -- `proc_funcdef` receives exactly one function's body in `raw_lines`, already separated from the `def` line and from surrounding code. No need to search for boundaries.

**2. Nesting-level tracking** -- `openindent`/`closeindent` markers are still present (they are not resolved until `reind_proc()`, which runs later). The `ind_change()` utility counts these markers to track nesting depth:

```python
def ind_change(inputstr):
    """Determine the change in indentation level (num opens - num closes)."""
    return inputstr.count(openindent) - inputstr.count(closeindent)
```

Combined with `def_regex` and `tco_disable_regex`, the algorithm can classify each scope:

- `level == 1` means direct function body
- `func_until_level` suppresses nested `def` scopes
- `disabled_until_level` suppresses `for`/`while`/`try`/`with` scopes

**3. Statement identification** -- Individual Coconut constructs have been compiled to Python by PyParsing (e.g., `x |> f` became `(f)(x)`), but control-flow keywords are preserved as-is since `return`, `raise`, `break`, and `continue` are the same in Python and Coconut. The `terminator_stmt_regex` reliably matches them:

```python
terminator_stmt_regex = compile_regex(r"\b(return|raise|break|continue)\b")
```

**4. Source line numbers** -- Line-number comments (e.g., `# 42`) are embedded in the compiled output during PyParsing but not yet resolved by `endline_repl()` (which runs later in post-processing). The `extract_line_num_from_comment()` utility extracts these, and `self.adjust()` maps them back to the original Coconut source:

```python
m = self.terminator_stmt_regex.match(base)
if m:
    raw_ln = extract_line_num_from_comment(comment)
    term_ln = self.adjust(raw_ln) if raw_ln is not None else None
    last_terminator = (m.group(1), term_ln)
```

### Established precedent

This is not a novel integration point. Two existing methods already operate on the same `raw_lines` at the same location inside `proc_funcdef`:

- **`detect_is_gen()`** -- iterates `raw_lines` to find `yield` statements, using the same `level`/`func_until_level` pattern to skip inner functions.
- **`transform_returns()`** -- iterates `raw_lines` to apply tail-call optimization and return transformation, using `level`, `func_until_level`, and `disabled_until_level` in exactly the same way.

`detect_unreachable_code()` follows the same pattern and sits alongside these methods as a natural peer.

---

## Timing Within Post-Processing

The detection must happen specifically in `deferred_code_proc()` -- not just anywhere in post-processing. The later steps progressively destroy the structural information that detection relies on:

| Post-processing step | What it does | What detection would lose |
|---|---|---|
| `deferred_code_proc()` | Resolves deferred function bodies | **Detection runs here** -- all information available |
| `passthrough_repl()` | Restores `\`-escaped Python | Minor -- but code may gain unexpected `return` keywords |
| `reind_proc()` | Converts `openindent`/`closeindent` to real whitespace | **Nesting tracking breaks** -- `ind_change()` no longer works |
| `endline_repl()` | Resolves line-number comments to final form | **Line number extraction breaks** -- `extract_line_num_from_comment()` no longer works |
| `str_repl()` | Restores string contents | `return` inside restored strings could cause false positives |
| `header_proc()` | Prepends runtime header | Unrelated |
| `polish()` | Final cleanup | Unrelated |

After `reind_proc()`, the indent markers that enable nesting-level tracking are gone. After `endline_repl()`, the line number comments that enable error reporting are gone. `deferred_code_proc()` is the **only** post-processing step where all four requirements are simultaneously satisfied.

---

## Summary

```
PRE-PROCESSING         PYPARSING              POST-PROCESSING

No function            Function bodies        Function bodies are
boundaries.            are DEFERRED --        fully available as
No syntax              not available          bounded raw_lines in
understanding.         as a unit.             proc_funcdef().
No AST.                No AST -- parse
                       actions emit           openindent/closeindent
Would need a           strings and            markers still present
second parser.         discard input.         for nesting tracking.

                                              Line number comments
                                              still intact for error
                                              reporting.

                                              Same pattern used by
                                              detect_is_gen() and
                                              transform_returns().

      X                      X                       OK
```

The unreachable code detection is implemented in `deferred_code_proc()` because it is the **unique moment** in the compilation pipeline when all four prerequisites converge: bounded function bodies, intact nesting markers, intact line numbers, and partially-compiled-but-still-structured code. Earlier is too early (no function boundaries); later is too late (structural markers destroyed).
