# How the Coconut Compiler Works and How Unreachable Code Detection Fits Into It

This document summarizes a conversation session exploring the Coconut compiler's architecture, its use of PyParsing, and where the unreachable code detection feature (issue #599) fits into the compilation pipeline.

---

## What is PyParsing?

PyParsing is a Python library for building parsers using a declarative, object-oriented approach. Instead of writing a formal grammar in a separate DSL (like ANTLR or yacc), you construct parsing rules directly in Python code by composing objects:

- `Literal("x")` — matches the exact string `"x"`
- `Keyword("if")` — matches a keyword (respects word boundaries)
- `expr1 + expr2` — sequential match (AND)
- `expr1 | expr2` — alternation (OR, first match wins)
- `ZeroOrMore(expr)` — repetition
- `.setParseAction(fn)` — attach a handler called when a rule matches

When parsing succeeds, handlers transform the matched tokens into output — in Coconut's case, Python code strings.

---

## How Coconut Uses PyParsing

Coconut's entire compiler frontend is a large PyParsing grammar defined as class attributes of the `Grammar` class in `grammar.py`. The `Compiler` class inherits from `Grammar`, giving it access to all parsers via `self`.

### Concrete Example: `"hello, world!" |> print`

**Grammar rules** (simplified from `grammar.py`):

```python
pipe = Literal("|>")
pipe_op = any_of(pipe, star_pipe, dubstar_pipe, ...)

pipe_item = labeled_group(comp_pipe_expr, "expr") + pipe_op
last_pipe_item = Group(comp_pipe_expr("expr"))

normal_pipe_expr_tokens = OneOrMore(pipe_item) + last_pipe_item
```

**Wiring to handler** (in `compiler.py`):

```python
cls.normal_pipe_expr <<= attach(cls.normal_pipe_expr_tokens, cls.method("pipe_handle"))
```

**Handler logic** (`pipe_handle` in `compiler.py`):

For `"hello, world!" |> print`, the handler produces:

```python
(print)("hello, world!")
```

Each rule's parse action immediately emits a Python code string — there is no separate AST. By the time PyParsing finishes, the entire file has been transformed bottom-up into one flat Python string.

---

## The Full Compilation Pipeline

```
Coconut source file
        |
        v
+-----------------------------------------------------+
|                   PRE-PROCESSING                     |
|  (compiler.py - Compiler.pre())                      |
|                                                      |
|  1. prepare         - normalize line endings         |
|  2. str_proc        - hide string contents behind    |
|                       placeholder markers            |
|  3. passthrough_proc - handle \ passthroughs         |
|  4. operator_proc   - handle custom operators        |
|  5. ind_proc        - convert indentation to         |
|                       openindent/closeindent markers |
+------------------------+----------------------------+
                         | clean, flat string with markers
                         v
+-----------------------------------------------------+
|                    PYPARSING                         |
|  (grammar.py - Grammar class attributes)             |
|                                                      |
|  file_parser.parseString(text)                       |
|                                                      |
|  PyParsing walks the string position by position,    |
|  trying to match grammar rules. Each match triggers  |
|  a parse action (handler) that returns a Python      |
|  code string. condense("".join) collapses all        |
|  tokens into one string.                             |
|                                                      |
|  Note: PyParsing does NOT think in lines. It works   |
|  character-by-character guided by rules.             |
|  ZeroOrMore(line) just means "keep matching `line`   |
|  at the current position until it fails."            |
+------------------------+----------------------------+
                         | compiled Python string (with markers still in it)
                         v
+-----------------------------------------------------+
|                  POST-PROCESSING                     |
|  (compiler.py - Compiler.post())                     |
|                                                      |
|  1. deferred_code_proc - resolve deferred output     |
|  2. passthrough repl   - restore \ passthroughs      |
|  3. reind_proc         - restore real indentation    |
|                          from openindent/closeindent |
|  4. endline_repl       - add line number comments    |
|  5. passthrough repl   - restore late passthroughs   |
|  6. str_repl           - restore string contents     |
|  7. header_proc        - prepend runtime header      |
|  8. polish             - strip trailing whitespace,  |
|                          ensure final newline        |
+------------------------+----------------------------+
                         |
                         v
              Final Python source file
```

### What the input to PyParsing looks like

Pre-processing replaces strings with markers, so PyParsing sees:

```
\x00strwrapper0\x00 |> print\n\x00strwrapper1\x00 |> print\n
```

### What PyParsing returns

A single flat string — all tokens joined by `condense` (`"".join`):

```
(print)(\x00strwrapper0\x00)\n(print)(\x00strwrapper1\x00)\n
```

### What post-processing produces

After restoring strings, adding line comments, and prepending the header:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# ... ~150KB runtime header ...

(print)("hello, world!")  #1 (line in Coconut source)
(print)("hello, world!")  #2 (line in Coconut source)
```

---

## Key Architectural Details

### `StartOfStrGrammar`

A thin wrapper class (`util.py`) that marks top-level parsers as "must match from position 0." Unwrapped in `prep_grammar()` right before actual parsing: stripped for `parseString` (which inherently starts at 0), or augmented with `StringStart()` for `scanString` (to prevent mid-string matching).

### `file_parser`

The top-level parser for compiling a full Coconut source file. Has two variants:

| Variant | When used | How it works |
|---|---|---|
| `raw_file_parser` | Default | Parses the entire file in one `parseString` call |
| `line_by_line_file_parser` | cPyparsing computation graph mode | A tuple of two parsers; the compiler calls each per line for cache reuse |

### Class inheritance

```python
class Compiler(Grammar, pickleable_obj):
```

`Grammar` defines all rules as class attributes. `Compiler` inherits from it, so `self.file_parser`, `self.pipe_handle`, etc. are all accessible.

---

## Where Unreachable Code Detection Fits

### The chosen location: `proc_funcdef` inside `deferred_code_proc`

`detect_unreachable_code` is called from `proc_funcdef` (`compiler.py:2808`), which is itself called from `deferred_code_proc` — the **first step of post-processing**.

During the PyParsing phase, function definitions are compiled but their body processing is **deferred** — wrapped in a marker and stored. It is only in `deferred_code_proc` that the compiler iterates the partially-compiled output, finds those deferred function wrappers, and calls `proc_funcdef` to finalize each one. This is where `detect_unreachable_code` runs.

```
POST-PROCESSING
      |
      +-- deferred_code_proc()          <-- first post-proc step
      |       |
      |       +-- finds a deferred funcdef marker
      |       |       |
      |       |       +-- calls proc_funcdef()
      |       |                 |
      |       |                 +-- strips the def line
      |       |                 +-- detect_unreachable_code(raw_lines)  <-- detection
      |       |                 +-- detect_is_gen(raw_lines)
      |       |                 +-- transform_returns(raw_lines)
      |       |
      |       +-- continues iterating lines...
      |
      +-- reind_proc()
      +-- endline_repl()
      +-- str_repl()
      +-- header_proc()
      +-- polish()
```

### Why this is the right place

1. **Complete function body available as a unit**: `proc_funcdef` has the entire function body in `raw_lines` with clear boundaries — no need to find where the function starts/ends.

2. **Established precedent**: `detect_is_gen` and `transform_returns` already iterate `raw_lines` at this exact point using the same pattern.

3. **Line number information is still intact**: Comment markers with line numbers haven't been resolved by `endline_repl` yet (that runs later in post-processing), so errors can still report correct source locations.

4. **Body is partially compiled but still structured**: Individual statements are transformed (pipes become `(f)(x)`) but `openindent`/`closeindent` markers are still raw, enabling reliable nesting-level tracking via `ind_change()`.

5. **Not too early, not too late**:
   - **Pre-processing** doesn't know about function structure — just a flat string.
   - **PyParsing** defers function body finalization — the body isn't fully available yet.
   - **`deferred_code_proc`** is the first moment where the complete, bounded function body is available for analysis, before later post-processing steps destroy the marker structure.

---

## Summary

The Coconut compiler works as a three-phase pipeline: pre-processing sanitizes the input, PyParsing matches grammar rules with handlers that emit Python code strings (deferring function body finalization), and post-processing restores hidden content and adds the runtime header. Unreachable code detection fits into the **post-processing phase**, specifically inside `deferred_code_proc` — the first post-processing step — which calls `proc_funcdef` to finalize each deferred function definition. This is the unique moment when the complete, bounded function body is available as a structured unit with line number markers still intact, and is the same point where the compiler already performs generator detection and return transformation.
