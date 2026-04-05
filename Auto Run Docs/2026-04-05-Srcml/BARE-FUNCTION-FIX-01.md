# Phase 01: Research & Test Case Design

**Goal:** Understand the exact parser behavior for bare function calls in C and create test cases that demonstrate the bug and expected output.

## Context

In C, `foo(x);` at statement level should be parsed as an **expression statement** containing a **function call** (`expr_stmt > expr > call`), not as a variable declaration. The srcML parser currently misclassifies this pattern.

The core issue is in `src/parser/srcMLParser.g` in the `pattern_check_core` rule around line 6660. The `set_type` logic at that location sets `type = VARIABLE` when it sees `NAME LPAREN NAME RPAREN TERMINATE` because:
- `type_count - specifier_count - template_count > 0` (1 > 0, since `foo` is counted as a type)
- `LA(1) == LPAREN && next_token() != RPAREN` (true, since `(x)` has a non-empty argument)

This causes the parser to treat `foo(x);` as a declaration of variable `x` of type `foo`.

## Key Files

- **Parser grammar:** `src/parser/srcMLParser.g`
  - `pattern_statements` rule: lines ~1391-1558
  - `pattern_check_core` rule: lines ~6105-6853 (especially `set_type` at lines 6660-6718)
  - `perform_call_check` rule: lines ~2771-2867
- **Existing C call tests:** `test/parser/testsuite/call_c.c.xml`
- **Existing C function tests:** `test/parser/testsuite/function_c.c.xml`
- **Test CMakeLists:** `test/parser/testsuite/CMakeLists.txt`

## Tasks

- [x] **Build srcml from the develop branch** to establish a working baseline. Check if a build system is already configured (look for an existing `build/` directory or CMake cache). If not, create a build directory and configure with CMake. Build just the `srcml` target. Record the path to the resulting binary.
  - **Note:** Built using Docker container `srcml/ubuntu:latest` (host lacks `libxslt1-dev`). CMake Release build configured in `/home/bill/agents/srcML/build/`. Binary at `build/bin/srcml`.

- [x] **Reproduce the bug with concrete examples.** Using the built `srcml` binary, test these C inputs and capture the actual output:
  1. `foo(x);` — bare call with single identifier argument
  2. `foo(x, y);` — bare call with multiple arguments
  3. `bar(1);` — bare call with literal argument (may already work since `1` can't be a type)
  4. `foo(x); bar(y);` — multiple bare calls
  5. Inside a function body: `void f() { foo(x); }` — bare call in function body context
  6. At file scope: `foo(x);` — bare call at global scope

  For each, document whether the output contains `<expr_stmt><expr><call>` (correct) or `<decl_stmt><decl>` / `<function_decl>` (incorrect). Save the results to a file at `test/parser/testsuite/bare_function_call_analysis.txt` for reference.
  - **Note:** All 6 tests plus 4 edge cases produce **correct** `<expr_stmt><expr><call>` output. The described bug does NOT reproduce on the current develop branch (commit ddab5fdde). Full analysis saved to `test/parser/testsuite/bare_function_call_analysis.txt`.

- [x] **Create the expected test file** `test/parser/testsuite/call_bare_c.c.xml` containing test cases for bare function calls in C. Follow the same XML format as existing test files like `call_c.c.xml`. Each `<unit>` block should contain the expected correct srcML output with `<expr_stmt><expr><call>` markup. Include cases for:
  1. `foo(x);` — single identifier argument
  2. `foo(x, y);` — multiple arguments
  3. `foo(x); bar(y);` — sequential bare calls
  4. Bare calls inside a function body: `void f() { foo(x); }`
  - **Note:** Created `test/parser/testsuite/call_bare_c.c.xml` with all 4 test cases. Added `LANGUAGE_C_ONLY call_bare_c` to `suite.txt`. All 4 tests pass via `srcml --parser-test` (0% failure rate, yellow=pass).
