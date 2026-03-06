# How Coconut Compiles a `.coco` File to Python

This document traces the full journey from running `coconut hello-world.coco` on the command line to getting a `hello-world.py` output file.

---

## Overview

```
coconut hello-world.coco
        │
        ▼
1. Entry point     main.py → Command().start()
        │
        ▼
2. CLI parsing     command/command.py → execute_args()
        │
        ▼
3. Compiler setup  compiler/compiler.py → Compiler.setup()
        │
        ▼
4. File dispatch   Command.compile_path() → compile_file() → compile()
        │
        ▼
5. Pre-processing  Compiler.pre()
   ├─ prepare()           Normalize line endings
   ├─ str_proc()          Replace strings/comments with markers
   ├─ passthrough_proc()  Handle \-passthroughs to raw Python
   ├─ operator_proc()     Register custom operator declarations
   └─ ind_proc()          Convert indentation to INDENT/DEDENT tokens
        │
        ▼
6. Parsing         PyParsing grammar (Grammar class)
   └─ Each grammar rule has an attached handler that transforms
      matched Coconut syntax into Python code strings
        │
        ▼
7. Post-processing Compiler.post()
   ├─ deferred_code_proc()   Emit deferred code (e.g. pattern-match helpers)
   ├─ base_passthrough_repl() Restore early passthroughs
   ├─ reind_proc()            Convert INDENT/DEDENT back to real whitespace
   ├─ endline_repl()          Fix end-of-line markers
   ├─ base_passthrough_repl() Restore \-passthroughs
   ├─ str_repl()              Restore original strings/comments
   ├─ header_proc()           Prepend the Coconut runtime header
   └─ polish()                Final cleanup (strip trailing whitespace, etc.)
        │
        ▼
8. Output          Write Python string to hello-world.py
```

---

## Step 1: Entry Point — `coconut/main.py`

When you run `coconut hello-world.coco`, Python invokes the `main()` function defined in `coconut/main.py`:

```python
def main():
    """Starts coconut."""
    Command().start()
```

`Command` lives in `coconut/command/command.py`. `start()` with `run=False` simply calls `self.cmd()`.

---

## Step 2: CLI Parsing — `Command.cmd()` and `execute_args()`

`cmd()` calls `arguments.parse_args()` (an `argparse`-based parser from `coconut/command/cli.py`) to turn the raw `sys.argv` into a namespace of flags and paths.

Key things determined at this stage:
- **Source path**: `hello-world.coco`
- **Destination path**: auto-generated as `hello-world.py` (by replacing the `.coco` extension with `.py`)
- **Target Python version**: empty string means "universal" (compatible with all Python versions)
- **Flags**: `--strict`, `--minify`, `--line-numbers`, `--no-tco`, etc.

After parsing, `execute_args()` calls `self.setup(...)` to configure the compiler, then calls `self.compile_path(source, dest, ...)`.

---

## Step 3: Compiler Setup — `Compiler.setup()`

`setup()` in `compiler/compiler.py` stores all compiler configuration parameters on `self`:

```python
self.target = ""       # Python target version
self.strict = False    # strict mode
self.minify = False    # minify output
self.line_numbers = True
self.keep_lines = False
self.no_tco = False    # tail-call optimization
self.no_wrap = False
self.pure = False
```

`Compiler` inherits from `Grammar` (in `compiler/grammar.py`), which defines all PyParsing grammar rules. The grammar is **built once at class definition time** using PyParsing combinators (e.g., `Literal`, `OneOrMore`, `Optional`, `Forward`).

---

## Step 4: File Dispatch — `compile_path → compile_file → compile()`

### `compile_path()`
Checks whether the source is a file or directory. For `hello-world.coco` it's a file, so it calls `compile_file()`.

### `compile_file()`
Determines the output path. Since no explicit destination was given, it auto-generates `hello-world.py` by stripping `.coco` and appending `.py`.

It also checks whether the file has already been compiled (via a hash stored in the output file header) to skip recompilation.

### `compile()`
This is the main method. It:
1. Reads the source file with `open(codepath, "r")`.
2. Checks a hash to skip recompilation if unchanged (`--force` bypasses this).
3. Calls `self.comp.parse_file(code, ...)` to do the actual compilation.
4. Writes the resulting Python string to `hello-world.py`.

```python
# Simplified from compile():
code = open("hello-world.coco").read()
compiled = self.comp.parse_file(code)
open("hello-world.py", "w").write(compiled)
```

---

## Step 5: Parsing Entry — `parse_file()` → `parse()`

`parse_file()` is a thin wrapper that picks the right parser and options for a standalone file:

```python
def parse_file(self, inputstring, addhash=True, **kwargs):
    use_hash = self.genhash(inputstring)  # MD5-based hash of the source
    return self.parse(inputstring, self.file_parser, {"nl_at_eof_check": True},
                      {"header": "file", "use_hash": use_hash}, ...)
```

The central `parse()` method orchestrates three phases:
1. **Pre-processing** (`self.pre(...)`)
2. **Parsing** (`parse(parser, pre_procd, ...)`) — PyParsing runs on the pre-processed string
3. **Post-processing** (`self.post(parsed, ...)`)

---

## Step 6: Pre-Processing — `Compiler.pre()`

`pre()` runs a sequence of **processor functions** (defined in `Compiler.preprocs`) over the source string one by one. Each processor takes a string and returns a transformed string.

```python
preprocs = [
    self.prepare,
    self.str_proc,
    self.passthrough_proc,
    self.operator_proc,
    self.ind_proc,
]
```

### 6a. `prepare()`
- Normalizes line endings (converts `\r\n` to `\n`).
- Stores line count for error reporting.
- Optionally strips leading/trailing whitespace.

### 6b. `str_proc()`
Scans the source character by character and **replaces all string literals and comments with placeholder markers**. This is critical: it prevents the grammar from accidentally matching Coconut syntax *inside* a string.

- A string like `"hello |> world"` gets replaced with something like `\x00STR_0\x00`.
- The original content is saved in a dictionary for later restoration.
- Comments (`# ...`) are similarly replaced.

### 6c. `passthrough_proc()`
Handles **Python passthrough syntax** (`\(...)` or `\\...`). This lets you embed raw Python code that Coconut should not transform. The passthrough content is wrapped in a special marker and preserved.

### 6d. `operator_proc()`
Scans for **custom operator declarations** (e.g., `operator @@`). When found, it registers the operator name in the compiler's state so the grammar can later recognize it. These lines are removed from the source.

### 6e. `ind_proc()`
Converts Python-style **significant indentation** into explicit `INDENT` and `DEDENT` marker tokens (stored as special Unicode characters: `openindent` and `closeindent`). This makes parsing much simpler since PyParsing doesn't natively understand indentation.

It also:
- Enforces that parenthetically-continued lines are folded into one line.
- Checks for balanced parentheses.
- Enforces trailing-whitespace rules in strict mode.

After all pre-processors run, the string is ready to be fed to the PyParsing grammar.

---

## Step 7: Grammar Parsing — `Grammar` class in `compiler/grammar.py`

The `Grammar` class (from which `Compiler` inherits) defines all grammar rules using **PyParsing combinators**. The rules are built at **class-definition time** (not at runtime per-parse), making subsequent parses fast.

### Grammar Construction
Grammar rules are defined declaratively as class attributes, for example:

```python
pipe = Literal("|>") | fixto(Literal("\u21a6"), "|>")
arrow = Literal("->") | fixto(Literal("\u2192"), "->")
comma = Literal(",")
```

More complex rules combine primitives:

```python
# Simplified: a pipeline expression like "x |> f |> g"
pipe_expr = expr + ZeroOrMore(pipe + expr)
```

### Handlers (Parse Actions)
Each grammar rule has a **handler** (parse action) attached via `attach()`. When PyParsing successfully matches a rule, it calls the handler with the matched tokens. The handler's job is to return a **Python code string** for that construct.

For example, the handler for `|>` (pipeline operator) transforms:
```coconut
x |> f
```
into:
```python
f(x)
```

And:
```coconut
range(10) |> map$(f) |> list
```
into:
```python
list(map(f, range(10)))
```

### Pattern Matching — `compiler/matching.py`
Complex constructs like `match` statements, pattern-matching function definitions (`def f(Some(x)):`) and destructuring assignments are handled by the `Matcher` class in `matching.py`.

When the grammar matches a `match` statement, its handler invokes `Matcher` to generate the equivalent Python `if/elif/else` chain with type checks, length checks, and variable bindings.

---

## Step 8: Post-Processing — `Compiler.post()`

After PyParsing returns a string of partially-transformed Python code (with markers still in place), `post()` runs a second sequence of processors:

```python
postprocs = [
    self.deferred_code_proc,
    self.base_passthrough_repl,  # early passthroughs
    self.reind_proc,
    self.endline_repl,
    self.base_passthrough_repl,  # \-passthroughs
    self.str_repl,
    self.header_proc,
    self.polish,
]
```

### 8a. `deferred_code_proc()`
Some code generation is **deferred** until after parsing (e.g., helper functions needed for pattern matching). This processor inserts them at the right locations.

### 8b. `reind_proc()`
Converts the `INDENT`/`DEDENT` markers back into real **Python whitespace indentation** (4-space by default).

### 8c. `endline_repl()`
Fixes end-of-line handling, converting internal newline markers back to real `\n` characters.

### 8d. `str_repl()`
**Restores** the original string literals and comments that were replaced by markers in `str_proc()`. This ensures strings appear verbatim in the Python output.

### 8e. `header_proc()`
Prepends the **Coconut runtime header** to the output. The header is generated by `compiler/header.py` from the template at `compiler/templates/header.py_template`.

The header contains:
- A shebang line and encoding declaration.
- A hash of the source (for future change detection).
- A `# Compiled with Coconut version X.Y.Z` comment.
- The Coconut runtime library: definitions of all built-in Coconut functions (`_coconut_pipe`, `_coconut_partial`, etc.), types (`MatchError`), and helpers needed by the compiled output.

For a **package** (directory compilation), the header instead imports from a shared `__coconut__.py` file to avoid duplicating the runtime in every file.

### 8f. `polish()`
Final cleanup: strips trailing whitespace and ensures the file ends with a single newline.

---

## Step 9: Output — Writing `hello-world.py`

Back in `compile()`, the fully compiled Python string is written to disk:

```python
with open("hello-world.py", "w") as f:
    f.write(compiled)
```

The compiler logs: `Compiled to hello-world.py`.

---

## End-to-End Example

Given `hello-world.coco`:
```coconut
greet = name -> "Hello, " + name + "!"
print(greet("World"))
```

After compilation, `hello-world.py` looks roughly like:
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

# Compiled with Coconut version X.Y.Z

# ... (Coconut runtime header) ...

greet = lambda name: "Hello, " + name + "!"
print(greet("World"))
```

---

## Key Files Reference

| File | Role |
|------|------|
| `coconut/main.py` | CLI entry point (`main()` and `main_run()`) |
| `coconut/command/command.py` | `Command` class: argument handling, file dispatch |
| `coconut/command/cli.py` | `argparse`-based CLI argument definitions |
| `coconut/compiler/compiler.py` | `Compiler` class: pre/post processors, `parse()` |
| `coconut/compiler/grammar.py` | `Grammar` class: all PyParsing grammar rules and handlers |
| `coconut/compiler/matching.py` | `Matcher` class: pattern-match code generation |
| `coconut/compiler/header.py` | Runtime header generation |
| `coconut/compiler/templates/header.py_template` | The actual Coconut runtime code |
| `coconut/constants.py` | Global constants: marker characters, targets, etc. |
| `coconut/exceptions.py` | Exception hierarchy |
| `coconut/terminal.py` | Logging (`logger`) |
