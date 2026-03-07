# How Coconut Compiles `.coco` to Python

## Overview

Coconut is a functional language that compiles to Python. The compilation is a one-pass process using PyParsing grammar with attached rewrite handlers.

---

## Entry Points

CLI compilation starts in `coconut/command/command.py` inside the `cmd()` method. After parsing CLI arguments and initializing the compiler, it calls:

```
compile_path(source, dest, ...)
```

### Call Chain

```
compile_path          # dispatches: is source a file or folder?
  └── compile_folder  # os.walk, finds all .coco files, calls compile_file per file
        └── compile_file  # resolves destination .py path
              └── compile   # reads source, hands off to Compiler
```

---

## `compile()` — `command.py:583`

This is the function that reads the `.coco` file and triggers compilation.

1. **Read source**: `code = opened.read()` — loads raw Coconut string.
2. **Hash check**: compares a hash of the source against the existing destination file. If unchanged and `--force` not passed, skips compilation.
3. **Submit job**: calls `submit_comp_job(..., "parse_file", code)` or `"parse_package"` depending on whether the file is part of a package.
4. **Callback**: `inner_callback` writes the compiled Python string to the `.py` destination file.

---

## Parallelism — `submit_comp_job()` — `command.py:678`

Handles whether compilation runs sequentially or in parallel:

- **Sequential** (`self.executor is None`):
  ```python
  callback(getattr(self.comp, method)(*args, **kwargs))
  ```
  Calls `self.comp.parse_file(code)` directly in the same process.

- **Parallel** (`self.executor` is a `ProcessPoolExecutor`):
  ```python
  future = self.executor.submit(multiprocess_wrapper(self.comp, method), *args, **kwargs)
  future.add_done_callback(callback_wrapper)
  ```
  Wraps the compiler call in `multiprocess_wrapper` (a pickleable callable) and submits it to a worker process. The callback fires when the future completes.

`self.executor` is set inside the `running_jobs()` context manager and is `None` by default. The default number of workers is `os.cpu_count()` (passed as `None` to `ProcessPoolExecutor`).

---

## `self.comp` — The Compiler Object

`self.comp` is an instance of `Compiler` (from `coconut/compiler/compiler.py`), initialized once in `setup()` before compilation begins. It holds all compiler state and grammar rules.

---

## `parse_file` / `parse_package` — `compiler.py:6042`

These are thin wrappers that:
1. Generate a hash string for the output file.
2. Call `self.parse(inputstring, self.file_parser, preargs, postargs)` with appropriate header options.

Other parse entry points exist for different contexts:

| Method | Used for |
|---|---|
| `parse_file` | Standalone `.coco` files |
| `parse_package` | Files inside a package (relative imports) |
| `parse_block` | REPL / inline code (stateful) |
| `parse_eval` | Expression evaluation |
| `parse_lenient` | Syntax highlighting / partial code |
| `parse_xonsh` | xonsh shell integration |

---

## `parse()` — `compiler.py:1809`

The core compilation method. Three stages:

### 1. Pre-processing — `self.pre(inputstring)`
Runs a chain of preprocessors (`self.preprocs`) on the raw source:
- Normalizes whitespace and line endings
- Replaces string literals with placeholders (so the grammar does not misparse their contents)
- Handles encoding declarations

### 2. Grammar Parsing — `parse(parser, pre_procd)` (from `util.py`)
Runs the PyParsing grammar against the preprocessed string. As grammar rules match, attached **parse actions** rewrite each Coconut construct into Python in-place. Examples:
- `x |> f` → `f(x)`
- `match x:` → `if`/`isinstance` checks
- `f$(x)` → `functools.partial(f, x)`

### 3. Post-processing — `self.post(parsed)`
Runs a chain of postprocessors (`self.postprocs`) on the raw parse output:
- Restores string literal placeholders to real strings
- Prepends the runtime header (imports for the Coconut runtime)
- Adds the hash comment to the output file
- Final formatting and cleanup

---

## `parse()` in `util.py:732`

```python
def parse(grammar, text, inner=None, eval_parse_tree=True, **kwargs):
    with parsing_context(inner):
        result = prep_grammar(grammar, for_scan=False).parseString(text, **kwargs)
        if eval_parse_tree:
            result = unpack(result)
        return result
```

- **`parsing_context(inner)`**: manages PyParsing's packrat/incremental cache. Saves and restores it for nested parse calls so they don't corrupt each other.
- **`prep_grammar(grammar)`**: unwraps `StartOfStrGrammar`, attaches tracing, marks as streamlined, calls `.parseWithTabs()` to preserve tabs.
- **`.parseString(text)`**: the actual PyParsing call. Matches grammar rules against the preprocessed source. As patterns match bottom-up, parse actions fire and rewrite tokens into Python fragments. Returns a nested `ParseResults` object.
- **`unpack(result)`**: evaluates lazy/deferred tokens via `final_evaluate_tokens`, then unwraps the outermost single-element `ParseResults` into a plain Python `str`.

---

## Full Pipeline Summary

```
.coco file
    │
    ▼
compile()               # read file, check hash
    │
    ▼
submit_comp_job()       # sequential or parallel dispatch
    │
    ▼
Compiler.parse_file()   # generate hash, select header type
    │
    ▼
Compiler.parse()
    ├── pre()           # normalize, replace string literals with placeholders
    ├── parse()         # PyParsing grammar rewrites Coconut → Python tokens
    │     ├── parsing_context  # cache safety
    │     ├── prep_grammar     # optimize grammar
    │     ├── parseString      # match rules, fire rewrite actions
    │     └── unpack           # evaluate lazy tokens → Python string
    └── post()          # restore strings, prepend header, add hash
    │
    ▼
.py file written by inner_callback
```
