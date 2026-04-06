# Reimplementing chibicc from Scratch

This document is a practical roadmap for building a C compiler with the same overall architecture as this repository. It is written as an incremental guide: start with a tiny compiler core, then extend it stage by stage until you approach the current feature set.

The target architecture in this codebase is x86-64 System V ABI on Linux, and the output format is assembly consumed by the system assembler and linker.

---

## 1. What you are building

A complete build in this repository is organized as:

1. **Driver** (`main.c`)  
   Parses CLI options and orchestrates preprocess/compile/assemble/link modes.
2. **Tokenizer** (`tokenize.c`)  
   Converts source text into a token stream with source locations.
3. **Preprocessor** (`preprocess.c`)  
   Handles macros, include files, and preprocessor conditionals.
4. **Parser + semantic model** (`parse.c`, `type.c`)  
   Builds ASTs and symbol objects, then annotates nodes with C types.
5. **Code generator** (`codegen.c`)  
   Emits x86-64 assembly for globals and functions.
6. **Support libraries** (`hashmap.c`, `strings.c`, `unicode.c`)  
   Provide data structures and UTF-8 helpers.

You can think of the central data flow as:

`text -> Token list -> preprocessed Token list -> AST/Object graph -> assembly -> object -> executable`

---

## 2. Recommended incremental implementation plan

The fastest way to reimplement this codebase is to keep each step executable and testable. Do not jump to full C11 immediately.

### Stage 0: Minimal executable pipeline

Goal: prove the plumbing works.

- Implement a tiny driver that accepts one `.c` file.
- Ignore preprocessing and parse only one integer literal.
- Emit assembly for `main` returning that integer.
- Run assembler/linker externally (`as`, `ld`, or `cc` wrapper).

At this stage you validate:
- subprocess invocation,
- temporary file handling,
- end-to-end compile to executable.

### Stage 1: Core lexer

Build `tokenize.c`-equivalent fundamentals:

- `Token` kind enum (ident, punct, number, string, EOF, etc.)
- token linked list with location metadata (`file`, `line`, `at_bol`, spacing)
- comment skipping (`//`, `/* */`)
- punctuator longest-match rules (`==`, `<=`, `>>=`, `##`, etc.)
- identifiers and keywords
- numeric preprocessing tokens (`TK_PP_NUM`) for later conversion
- string/char literals with escapes
- robust diagnostics (`error_at`, `error_tok`)

Keep source normalization early:

- newline canonicalization,
- backslash-newline splicing,
- universal character escape conversion,
- optional BOM skip.

### Stage 2: Minimal parser and AST

Define AST node kinds and write a recursive descent parser for:

- expressions (`+ - * /`, unary, assignment),
- return statements,
- compound blocks,
- function definition for `int main()`.

Add enough AST generation so codegen can emit stack-machine-like expression evaluation.

### Stage 3: Type system backbone

Introduce a `Type` model with primitive kinds and pointers:

- integer/float base types,
- pointer type construction,
- arrays/functions structs in type model shape (even if not fully used yet),
- type attachment pass (`add_type`) over AST.

Implement usual arithmetic conversion early; it simplifies later operators and casts.

### Stage 4: Basic code generation

Create `codegen.c` equivalent that emits:

- function prologue/epilogue,
- stack slots for locals,
- integer expression evaluation,
- loads/stores for lvalues,
- control flow labels (`if`, loops),
- `return`.

Keep code generation straightforward and explicit. This codebase favors readability over dense abstractions.

### Stage 5: Full declaration parser and symbols

Expand parser to support:

- declaration specifiers,
- declarators (pointer/function/array nesting),
- local/global variables,
- function parameters,
- storage class (`static`, `extern`, `typedef`, thread local),
- initializers.

Introduce `Obj` symbol records for globals/functions/locals, matching this repository’s design.

### Stage 6: Real preprocessor

Implement an independent preprocessor pass:

- object-like and function-like macros,
- hidesets to prevent recursive re-expansion,
- `#include`, include search paths, include guards, `#pragma once`,
- conditional directives (`#if/#ifdef/#ifndef/#elif/#else/#endif`),
- predefined macros and builtin macro handlers (`__FILE__`, `__LINE__`, etc.),
- post-preprocess token normalization:
  - pp-number conversion,
  - keyword classification,
  - adjacent string literal concatenation.

This is one of the highest complexity parts; build it in sub-steps and test aggressively.

### Stage 7: Driver parity (`main.c` behavior)

Implement gcc-like driver modes:

- `-E`: preprocess only,
- `-S`: compile to assembly,
- `-c`: compile + assemble,
- default: compile + assemble + link.

Add option parsing and forwarding through a separate internal frontend mode (this repo uses `-cc1`).

This split keeps the frontend reusable while the driver handles toolchain integration.

### Stage 8: ABI-compliant calls and aggregates

Match x86-64 SysV calling convention details:

- register vs stack argument passing,
- struct/union passing/return (including float aggregate classification),
- variadic handling (`va_area`/register save area logic),
- local alignment rules.

This stage is needed to compile real software correctly.

### Stage 9: C11 and GCC-extension completion

Incrementally add the long tail:

- structs/unions/enums/bitfields,
- VLAs and `alloca`,
- designated initializers,
- `_Atomic` and atomic expressions used by this compiler,
- statement expressions and labels-as-values,
- `_Thread_local`,
- `typeof`, `asm`, and selected GCC extensions present in this repo.

### Stage 10: Self-host and regression hardening

Use the test strategy from this repository:

- per-feature tests under `test/*.c`,
- driver-level integration tests via `test/driver.sh`,
- bootstrap/stage2 build (compile compiler with itself and retest).

A practical milestone is passing `make test` then `make test-stage2`.

---

## 3. How this repository maps to the stages

- **Driver and orchestration:** `/home/runner/work/chibicc/chibicc/main.c`
- **Public compiler data model:** `/home/runner/work/chibicc/chibicc/chibicc.h`
- **Lexing and diagnostics:** `/home/runner/work/chibicc/chibicc/tokenize.c`
- **Macro engine and directives:** `/home/runner/work/chibicc/chibicc/preprocess.c`
- **Grammar, symbols, initializers, constant folding:** `/home/runner/work/chibicc/chibicc/parse.c`
- **Type propagation and compatibility:** `/home/runner/work/chibicc/chibicc/type.c`
- **x86-64 emission:** `/home/runner/work/chibicc/chibicc/codegen.c`
- **Helpers:** `/home/runner/work/chibicc/chibicc/hashmap.c`, `/home/runner/work/chibicc/chibicc/strings.c`, `/home/runner/work/chibicc/chibicc/unicode.c`
- **Build/tests:** `/home/runner/work/chibicc/chibicc/Makefile`, `/home/runner/work/chibicc/chibicc/test`

---

## 4. Suggested implementation order inside each module

If you are re-writing from zero, a useful order is:

1. **`chibicc.h` first**: freeze core structs (`Token`, `Type`, `Node`, `Obj`) early.
2. **`tokenize.c` next**: everything depends on reliable tokens and diagnostics.
3. **Small `parse.c` + small `codegen.c`**: get executable output quickly.
4. **`type.c`**: move type logic out of parser as complexity grows.
5. **`preprocess.c`**: add after basic compiler works; integrate before parsing.
6. **Expand parser/codegen feature-by-feature** while tests grow.
7. **Finalize driver parity** and system linker integration last.

---

## 5. Design choices worth copying

This repository intentionally optimizes for readability and incremental learning:

- simple data structures over abstract frameworks,
- explicit recursive-descent functions per grammar area,
- clear pipeline boundaries,
- no optimization pass yet (prioritize correctness first),
- allocate-and-exit memory strategy for compiler process lifetime.

These choices are useful when reimplementing because they reduce incidental complexity while the language surface is still expanding.

---

## 6. Practical validation checkpoints

Use these checkpoints while reimplementing:

1. **Can compile one return statement program.**
2. **Can parse and execute arithmetic/control flow tests.**
3. **Can preprocess includes/macros for simple headers.**
4. **Can compile all feature tests in `test/` with host compiler link.**
5. **Can self-compile to stage2 and pass stage2 tests.**

Treat each checkpoint as a release point before adding new language features.

---

## 7. Scope note

This guide mirrors the architecture and implementation strategy of this repository; it is not a generic optimizing compiler textbook path. If your goal is a production compiler with aggressive optimization and retargetable backends, treat this as a frontend-and-codegen foundation to build on, not the final architecture.
