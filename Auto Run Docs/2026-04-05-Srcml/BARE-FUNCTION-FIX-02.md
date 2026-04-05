# Phase 02: Parser Grammar Fix

**Goal:** Modify the srcML parser grammar so that bare function calls like `foo(x);` in C are parsed as expression statements with call markup, not as declarations.

## Context

The bug is in `src/parser/srcMLParser.g` in the `pattern_check_core` rule. When the parser encounters `foo(x);`, the `set_type` logic at lines ~6660-6718 incorrectly classifies it as `VARIABLE` because:
- `foo` is counted as a type token (`type_count = 1`)
- `(x)` matches the `LA(1) == LPAREN && next_token() != RPAREN` condition
- Combined, these satisfy the VARIABLE predicate

The fix needs to distinguish between:
- `int(x);` or `mytype(x);` — actually a declaration (where the first token IS a known type)
- `foo(x);` — a function call (where the first token is just an identifier)

The parser already has a `MODE_FUNCTION_BODY` check at lines 6820-6825 that throws an exception (bailing out of VARIABLE classification) when inside a function body. But this doesn't cover file scope or other contexts.

## Key Decision Point

The fix likely needs to happen in one of these areas:
1. **`set_type` predicate (lines 6660-6718):** Add a condition that prevents VARIABLE classification when inside a function body or when the name doesn't match a known type. The `MODE_FUNCTION_BODY` check at line 6822 already partially handles this — the fix may need to extend that logic.
2. **`pattern_check` flow (lines 5873-6097):** Modify the flow so that bare calls are detected before reaching `pattern_check_core`.
3. **`perform_call_check` integration:** Ensure the call check runs and takes precedence in ambiguous cases for C.

## Tasks

- [x] **Analyze the exact token flow for `foo(x);` in the parser.** Read and trace through the grammar rules to determine exactly where `foo(x);` gets misclassified. Focus on:
  - What values `type_count`, `specifier_count`, `real_type_count` have when processing `foo(x);`
  - Whether the issue is in `set_type` at line 6660 or in `pattern_check_core`'s function detection at line 6795
  - How `perform_call_check` at line 1544 interacts with the flow — does it ever get called for bare calls?
  - How the `MODE_FUNCTION_BODY` exception at line 6820 works and why it doesn't cover all cases

  Document your analysis as comments in your commit message.

  **Analysis Result — Bug does NOT exist on current develop branch:**

  Token flow trace for `foo(x);` (tokens: NAME(foo), LPAREN, NAME(x), RPAREN, TERMINATE):

  1. **Type token loop (lines 6165-6586):** Iteration 1 consumes `foo` via `compound_name` (line 6533). `type_count` → 1. Iteration 2 sees LPAREN — the loop predicate at 6167 passes (LPAREN ≠ LBRACKET), and LPAREN is in `argument_token_set` (line 66 of `srcml_bitset_token_sets.hpp`). However, **no alternative in the loop body matches LPAREN** — it's not a specifier, template, class keyword, compound_name, etc. This causes a `NoViableAltException`, which propagates up to the catch block in `pattern_check` (line 5905).

  2. **At exception time:** `type_count = 1` (from the single `foo` token), `type = NONE`, `specifier_count = 0`. The critical post-loop adjustments at lines 6603-6627 (including `type_count = type_count > 1 ? type_count - 1 : 0`) are **never reached** because the exception aborts `pattern_check_core`.

  3. **In the catch block (line 5905-5908):** The condition `type == VARIABLE && type_count == 0` is false (type is NONE), so no adjustment is made.

  4. **Post-catch processing (lines 5911-6097):** None of the type reclassification conditions match — `type` stays NONE/0.

  5. **Statement dispatch (lines 1470-1558):** With `stmt_type == NONE`, all specific statement type checks fail. The flow reaches `expression_statement[type, call_count]` at line 1557, which correctly parses `foo(x);` as `<expr_stmt><expr><call>`.

  **Key insight:** The `set_type[VARIABLE]` at line 6660 is never reached for `foo(x);` because the parser throws an exception during the type-token loop when it encounters LPAREN (which is in `argument_token_set` but doesn't match any loop alternative). The described bug scenario where `type_count=1` satisfies the VARIABLE predicate cannot occur because the VARIABLE check at line 6660 requires the type-token loop to complete normally (reaching line 6627 which would set `type_count = 0` for a single-token case anyway).

  **`perform_call_check` (line 1544):** This is called in the statement dispatch for macro detection (`type == MACRO`). For bare calls like `foo(x);` it is NOT called because `stmt_type` is NONE, falling through directly to `expression_statement`.

  **`MODE_FUNCTION_BODY` exception (line 6820):** This is a safety net inside `pattern_check_core` that throws an exception if something is classified as VARIABLE inside a function body with a TERMINATE follow token. It's never triggered for `foo(x);` because the parser never reaches that point (it throws earlier in the type-token loop).

  **Verified:** `echo 'foo(x);' | srcml --language C` produces correct `<expr_stmt><expr><call>` output. All 4 bare call test cases pass. All regression tests pass (call_c, function_c, function_decl_c, decl_simple_c, expression_c — 0% failure across 47 test units).

- [x] **Implement the grammar fix in `src/parser/srcMLParser.g`.** Based on the analysis above, modify the grammar so that `foo(x);` in C is correctly parsed as an expression statement containing a function call. The fix should:
  - Not break any existing tests (declarations like `int x;`, `int(x);`, function declarations, function definitions must still work)
  - Handle bare calls both inside function bodies and at file scope
  - Only apply to C (and potentially C++) — do not change Java/C#/Python behavior
  - Be minimal — avoid large refactors of the grammar

  **Result: No grammar fix needed.** The parser already correctly handles `foo(x);` as an expression statement with call markup. The described bug does not exist on the current develop branch. See analysis above for the detailed token flow explanation.

- [x] **Verify the fix by running the bare function call test cases** from Phase 01 (`test/parser/testsuite/call_bare_c.c.xml`) against the modified parser. Build the parser and test each case. If any cases fail, adjust the fix. Also run a broader set of existing C tests to check for regressions:
  - `test/parser/testsuite/call_c.c.xml`
  - `test/parser/testsuite/function_c.c.xml`
  - `test/parser/testsuite/function_decl_c.c.xml`
  - `test/parser/testsuite/decl_simple_c.c.xml`
  - `test/parser/testsuite/expression_c.c.xml`

  **Result: All tests pass.** Bare call tests: 4/4 pass. Regression tests: call_c (4/4), function_c (14/14), function_decl_c (4/4), decl_simple_c (19/19), expression_c (6/6) — total 47/47 tests pass with 0% failure rate.
