# Reimplementing chibicc from Scratch (Beginner Walkthrough)

This guide is for beginners in compiler engineering who want to rebuild a compiler like this repository from nothing.

You are not expected to understand every C11 corner case before starting. Instead, you build a tiny compiler first, keep it running, and then add one capability at a time.

All references below point to this repository for concrete implementation tips:

- `/home/runner/work/chibicc/chibicc/chibicc.h`
- `/home/runner/work/chibicc/chibicc/main.c`
- `/home/runner/work/chibicc/chibicc/tokenize.c`
- `/home/runner/work/chibicc/chibicc/preprocess.c`
- `/home/runner/work/chibicc/chibicc/parse.c`
- `/home/runner/work/chibicc/chibicc/type.c`
- `/home/runner/work/chibicc/chibicc/codegen.c`
- `/home/runner/work/chibicc/chibicc/Makefile`
- `/home/runner/work/chibicc/chibicc/test`

---

## 0) Mental model: what this compiler does

At a high level:

1. Read C source text.
2. Tokenize it (characters -> tokens).
3. Preprocess macros/includes (`#define`, `#include`, `#if`, etc.).
4. Parse tokens into an AST and symbol graph.
5. Add/verify type information.
6. Emit x86-64 assembly.
7. Assemble and link to produce an executable.

In this repo, the major boundaries are clean and visible:

- Driver/tool invocation: `main.c`
- Frontend lexing/preprocessing/parsing/typing: `tokenize.c`, `preprocess.c`, `parse.c`, `type.c`
- Backend code emission: `codegen.c`
- Shared core structs and APIs: `chibicc.h`

Beginner tip: keep these boundaries in your own rewrite. Debugging is much easier when each stage has a clear input and output.

---

## 1) Build the smallest possible compiler first

### Goal

Produce an executable for a trivial program like:

```c
int main() { return 42; }
```

### What to implement first

- A tiny driver (`main` function) that:
  - reads one input file,
  - emits one assembly file,
  - invokes system tools to assemble/link.

### Codebase references

- Driver option and subprocess structure:
  - `parse_args`, `run_subprocess`, `run_cc1`, `assemble`, `run_linker` in `/home/runner/work/chibicc/chibicc/main.c`
- Build flow reference:
  - `test`, `test-stage2`, and compile/link recipes in `/home/runner/work/chibicc/chibicc/Makefile`

### Beginner checkpoint

If you can compile one function that returns a constant, stop and verify end-to-end before adding parsing complexity.

---

## 2) Define your core data model early (`chibicc.h` equivalent)

### Why this matters

Most implementation pain in a compiler comes from unstable core structs. Freeze the shape of tokens, AST nodes, symbols, and types early.

### Minimum structs to define

- `Token`
- `Node` (AST)
- `Obj` (variables/functions/symbol entries)
- `Type`
- `Member` (for struct/union members)

### Codebase references

In `/home/runner/work/chibicc/chibicc/chibicc.h`, study:

- Token model:
  - `TokenKind` enum (`TK_IDENT`, `TK_PUNCT`, `TK_KEYWORD`, `TK_STR`, `TK_NUM`, `TK_PP_NUM`, `TK_EOF`)
  - `struct Token` fields (`loc`, `len`, `file`, `line_no`, `at_bol`, `has_space`, `hideset`)
- Symbol model:
  - `struct Obj` for locals, globals, and functions
- AST model:
  - `NodeKind` enum (large set; add in phases in your own compiler)
  - `struct Node` payload fields for expressions/statements/control flow
- Type model:
  - `TypeKind` enum and `struct Type`
  - `base` pointer for pointer/array behavior

### Beginner tip

Use one broad `Node` struct first (as this codebase does). You can optimize memory layout later.

---

## 3) Implement diagnostics before deep parsing

### Why first

Good errors speed up every future stage.

### What to build

- fatal error API (`error`, `error_at`, `error_tok`)
- warning API (`warn_tok`)
- source line and caret printing

### Codebase references

In `/home/runner/work/chibicc/chibicc/tokenize.c`:

- `error`
- `verror_at`
- `error_at`
- `error_tok`
- `warn_tok`

These functions are used from all over the compiler. Copy this architecture early.

---

## 4) Build tokenizer (lexer) in practical substeps

Start with a linked-list token stream and add token classes incrementally.

### 4.1 Token list and helpers

Implement:

- token creation (`new_token`-style helper)
- token comparison (`equal`)
- mandatory token consume (`skip`)
- optional consume (`consume`)

Reference: `/home/runner/work/chibicc/chibicc/tokenize.c` (`equal`, `skip`, `consume`, `new_token`)

### 4.2 Identifier and punctuator scanning

Implement:

- `read_ident` style identifier scanner
- longest-match punctuator scanner (`read_punct`)

Reference functions:

- `read_ident`
- `read_punct`
- `startswith`

### 4.3 Literals

Implement literals in this order:

1. integer numbers,
2. string literals with escapes,
3. character literals,
4. floating literals.

Reference functions:

- `read_escaped_char`
- `string_literal_end`
- string literal readers in `/home/runner/work/chibicc/chibicc/tokenize.c`

### 4.4 Source normalization (important and often skipped by beginners)

Do these before normal lexing:

- normalize newline forms (`canonicalize_newline`)
- remove backslash-newline continuations (`remove_backslash_newline`)
- convert universal chars (`convert_universal_chars`)

Reference functions:

- `canonicalize_newline`
- `remove_backslash_newline`
- `read_universal_char`
- `convert_universal_chars`

### 4.5 Post-tokenization conversion

This codebase tokenizes some numbers first as preprocessing-number and then converts later:

- `convert_pp_int`
- `convert_pp_number`
- `convert_pp_tokens`

Reference: `/home/runner/work/chibicc/chibicc/tokenize.c`

### Beginner checkpoint

Add a mode to print tokens (see `print_tokens` in `main.c`) and verify punctuation/spacing behavior on real source files.

---

## 5) Parse expressions and statements with recursive descent

This repository’s parser is intentionally explicit and readable: one function per grammar area.

### 5.1 Start with expression precedence chain

Implement in precedence order:

- `assign`
- `conditional`
- `logor`
- `logand`
- `bitor`
- `bitxor`
- `bitand`
- `equality`
- `relational`
- `shift`
- `add`
- `mul`
- `cast`
- `unary`
- `postfix`
- `primary`

Reference: function set in `/home/runner/work/chibicc/chibicc/parse.c`

### 5.2 Then statements

Implement:

- expression statement,
- return statement,
- block statement,
- `if`/`for`/`while`/`switch`/`goto` progressively.

Reference functions:

- `stmt`
- `compound_stmt`
- `expr_stmt`

### 5.3 Node construction style

Keep dedicated constructors:

- `new_node`
- `new_binary`
- `new_unary`
- `new_num`
- `new_var_node`

Reference: `/home/runner/work/chibicc/chibicc/parse.c`

### Beginner tip

Do not try parser generators at this stage. A hand-written recursive descent parser is easier to debug while learning.

---

## 6) Add scope and symbol tables

You need nested scopes for locals/typedef names/tags.

### What to implement

- scope stack with push/pop
- variable and tag lookup
- symbol creation for local/global variables

### Codebase references

In `/home/runner/work/chibicc/chibicc/parse.c`:

- scope model: `Scope`, `VarScope`
- scope operations: `enter_scope`, `leave_scope`
- lookup: `find_var`, `find_tag`
- symbol creation: `new_var`, `new_lvar`, `new_gvar`, `push_scope`

### Beginner checkpoint

Test variable shadowing and block scopes (`{ int x; { int x; } }`).

---

## 7) Add type system and type propagation

Keep parsing and typing conceptually separate. Parse first, then annotate AST with types.

### What to implement

- primitive builtin types (`int`, `char`, `long`, float types)
- pointer and array constructors
- function types
- type compatibility checks
- usual arithmetic conversions
- AST type-attachment pass

### Codebase references

In `/home/runner/work/chibicc/chibicc/type.c`:

- canonical type singletons:
  - `ty_void`, `ty_bool`, `ty_char`, `ty_int`, `ty_long`, etc.
- constructors:
  - `pointer_to`, `array_of`, `func_type`, `vla_of`, `struct_type`, `enum_type`
- reasoning helpers:
  - `is_integer`, `is_flonum`, `is_numeric`, `is_compatible`
- conversion logic:
  - `get_common_type`, `usual_arith_conv`
- central pass:
  - `add_type`

### Beginner tip

Type errors should include token location (`error_tok`) so users see the exact bad expression.

---

## 8) Implement declaration parsing (hard but essential)

C declarators are one of the most confusing parts for beginners.

### Build this in layers

1. declaration specifiers (`int`, `unsigned`, `static`, etc.),
2. pointer chains (`*`, qualifiers),
3. suffixes (arrays/functions),
4. full declarator assembly.

### Codebase references

In `/home/runner/work/chibicc/chibicc/parse.c`:

- `declspec`
- `pointers`
- `type_suffix`
- `declarator`
- `abstract_declarator`
- `typename`
- `func_params`
- `array_dimensions`

### Beginner tip

Do not optimize this logic early. Keep it verbose and test each declarator form independently.

---

## 9) Add initializers (locals and globals)

Initializers are tree-shaped (especially arrays/structs/unions/designators).

### What to implement

- initializer tree structure
- array/struct/union initialization
- designated initializers
- local initialization lowering to assignment AST
- global initialization lowering to data + relocations

### Codebase references

In `/home/runner/work/chibicc/chibicc/parse.c`:

- data model:
  - `Initializer`, `InitDesg`
- builders:
  - `new_initializer`, `initializer`, `initializer2`
- aggregate handling:
  - `array_initializer1`, `array_initializer2`
  - `struct_initializer1`, `struct_initializer2`
  - `union_initializer`
- designators:
  - `designation`, `array_designator`, `struct_designator`
- output:
  - `lvar_initializer`
  - `gvar_initializer`
  - relocation-writing helpers around `write_gvar_data`

### Beginner checkpoint

Verify nested initializers and mixed designated/non-designated forms.

---

## 10) Build code generation for x86-64 (SysV ABI)

Start minimal (integers/locals/return), then add full ABI behavior.

### 10.1 Core expression/statement emission

Implement:

- expression emitter (`gen_expr`)
- statement emitter (`gen_stmt`)
- lvalue address generation (`gen_addr`)
- load/store split (`load`, `store`)

Reference: `/home/runner/work/chibicc/chibicc/codegen.c`

### 10.2 Function frame and locals

Implement:

- local offset assignment
- function prologue/epilogue
- stack alignment

Reference functions:

- `assign_lvar_offsets`
- `emit_text`
- `align_to`

### 10.3 Data emission

Implement:

- global variables and string literal data sections
- relocations for global addresses

Reference:

- `emit_data` in `/home/runner/work/chibicc/chibicc/codegen.c`

### 10.4 Function calls and ABI details

Implement:

- integer/float register arguments,
- stack arguments,
- variadic support,
- struct return conventions.

Reference functions:

- `push_args`, `push_args2`
- `copy_ret_buffer`, `copy_struct_reg`, `copy_struct_mem`
- register arrays (`argreg8`, `argreg16`, `argreg32`, `argreg64`)

### Beginner tip

When behavior is wrong, dump assembly and compare against GCC/Clang for tiny reproducer programs.

---

## 11) Implement preprocessor as an independent pass

This is a separate language layer with different rules than normal C parsing.

### 11.1 Macro representation and storage

Implement:

- macro table keyed by name,
- object-like and function-like forms,
- macro parameters and varargs.

Reference structures:

- `Macro`, `MacroParam`, `MacroArg` in `/home/runner/work/chibicc/chibicc/preprocess.c`

### 11.2 Expansion safety with hidesets

Implement hidesets to prevent infinite recursive expansion.

Reference functions:

- `new_hideset`
- `hideset_union`
- `hideset_intersection`
- `hideset_contains`
- `add_hideset`
- `expand_macro`

### 11.3 Directive engine

Implement:

- `#define`, `#undef`,
- `#include` and include path search,
- `#if/#ifdef/#ifndef/#elif/#else/#endif`,
- `#line`,
- `#pragma once` and include guard detection.

Reference functions:

- definition/args:
  - `read_macro_params`, `read_macro_definition`, `read_macro_arg_one`
- include:
  - `read_include_filename`, `include_file`, `detect_include_guard`
- conditional stack:
  - `CondIncl`, `push_cond_incl`, `skip_cond_incl`, `skip_cond_incl2`
- engine:
  - `preprocess2`

### 11.4 Builtin macros and final token cleanup

Reference functions:

- builtins:
  - `file_macro`, `line_macro`, `counter_macro`, `timestamp_macro`, `base_file_macro`
- postprocessing:
  - `join_adjacent_string_literals`

### Beginner checkpoint

Pass simple macro tests before complex nested conditional tests. Preprocessor bugs can cascade into parser confusion.

---

## 12) Build a real command-line driver

A compiler executable is both frontend and toolchain orchestrator.

### What to support

- preprocessing only (`-E`)
- assembly output (`-S`)
- object output (`-c`)
- full link path (default)
- include and macro options (`-I`, `-D`, `-U`, `-include`)

### Codebase references

In `/home/runner/work/chibicc/chibicc/main.c`:

- option parser:
  - `parse_args`, `take_arg`, `parse_opt_x`
- cc1 split:
  - `run_cc1`, `cc1`
- dependency helpers:
  - `print_dependencies`, `in_std_include_path`
- system integration:
  - `assemble`, `run_linker`
- temp file lifecycle:
  - `create_tmpfile`, `cleanup`

### Beginner tip

Keep frontend logic and subprocess logic separate as early as possible.

---

## 13) Add advanced C features in practical order

After baseline correctness, add features in small slices:

1. Struct/union/enum parsing and layout.
2. Bitfields.
3. Function pointers and variadics.
4. Designated initializers.
5. VLAs and `alloca`.
6. Thread-local storage and atomics.
7. GCC extensions you need (`typeof`, statement expressions, labels-as-values, `asm`).

### Codebase pointers

- struct/union/tag parsing:
  - `struct_union_decl`, `struct_decl`, `union_decl`, `struct_members` in `parse.c`
- enums:
  - `enum_specifier` in `parse.c`
- generic selection:
  - `generic_selection` in `parse.c`
- atomics and asm nodes:
  - `ND_CAS`, `ND_EXCH`, `ND_ASM` in `chibicc.h` and handling in `parse.c`/`codegen.c`

---

## 14) Testing strategy you should copy

### Repository test flow

In `/home/runner/work/chibicc/chibicc/Makefile`:

- `make test`:
  - builds compiler,
  - compiles feature tests in `/home/runner/work/chibicc/chibicc/test`,
  - runs `test/driver.sh` integration checks.
- `make test-stage2`:
  - self-host style stage2 compiler build,
  - reruns test corpus with stage2 binary.

### Suggested beginner workflow

For each new feature:

1. add/enable one tiny test,
2. compile with your compiler,
3. run executable and compare expected output,
4. only then expand feature coverage.

---

## 15) Common beginner failure modes (and where to look)

1. **Tokenizer offsets wrong** -> inspect `Token.loc`, `len`, and line number assignment (`add_line_numbers` in `tokenize.c`).
2. **Parser loops or consumes wrong token** -> check `skip`/`consume` usage patterns.
3. **Type confusion in binary operators** -> check `usual_arith_conv` and cast insertion.
4. **Lvalue/rvalue bugs** -> inspect `gen_addr`, `load`, and `store`.
5. **Call ABI mismatch** -> inspect argument placement code (`push_args*`) and return-buffer handling.
6. **Macro recursion explosion** -> verify hideset operations and expansion conditions.
7. **Driver works for `-S` but not default link** -> inspect linker argument construction in `run_linker`.

---

## 16) Suggested reading order inside this repo (for learners)

1. `/home/runner/work/chibicc/chibicc/chibicc.h` (understand data model first)
2. `/home/runner/work/chibicc/chibicc/tokenize.c` (input and diagnostics)
3. `/home/runner/work/chibicc/chibicc/parse.c` (grammar and AST)
4. `/home/runner/work/chibicc/chibicc/type.c` (semantic typing)
5. `/home/runner/work/chibicc/chibicc/codegen.c` (assembly emission)
6. `/home/runner/work/chibicc/chibicc/preprocess.c` (macro engine)
7. `/home/runner/work/chibicc/chibicc/main.c` (driver and toolchain orchestration)
8. `/home/runner/work/chibicc/chibicc/test` and `/home/runner/work/chibicc/chibicc/Makefile` (how correctness is enforced)

---

## 17) Final milestone definition

You are close to “reimplemented chibicc-class compiler” when all are true:

1. You pass your equivalent of `make test`.
2. You can self-compile into a stage2 compiler and re-pass tests (`make test-stage2` equivalent).
3. You support enough C11 + selected GCC extensions to compile non-trivial external code.
4. Error messages still point users to exact source locations.

At that point, you can decide whether to keep the educational architecture (as this repository does) or start introducing optimizer/backend abstractions.
