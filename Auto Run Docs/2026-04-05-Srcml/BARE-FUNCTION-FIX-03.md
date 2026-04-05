# Phase 03: Regression Testing & Commit

**Goal:** Run the full parser test suite to verify no regressions, fix any issues, and commit the changes.

## Tasks

- [x] **Run the full parser test suite.** Build the test target and run CTest from the build directory. The test suite is at `test/parser/` and uses CMake/CTest. Run all parser tests and capture results. If there are failures, categorize them as:
  1. Pre-existing failures (not related to our change)
  2. Regressions caused by our change

  For any regressions, fix them by adjusting the grammar change in `src/parser/srcMLParser.g` or updating test expectations if the new behavior is correct.

  **Result: All 5169 parser tests pass with 0% failure rate.** Breakdown by language: C (158), C# (498), C++ (3048), Java (233), Objective-C (193), Python (1039). Zero errors, zero regressions. The bare function call test (`call_bare_c`) passes all 4 test units.

- [x] **Register the new test file in the test infrastructure.** Ensure `test/parser/testsuite/call_bare_c.c.xml` is picked up by the test system. Check `test/parser/testsuite/CMakeLists.txt` to see if new `.xml` files are auto-discovered via `file(GLOB PARSE_TESTS *.xml)` (they appear to be at line 14). If additional registration is needed (e.g., `setlanguage` macros), add it. Verify the new test runs and passes.

  **Result: Already registered.** The test file is auto-discovered by `file(GLOB PARSE_TESTS *.xml)` at CMakeLists.txt line 14, and is registered in `suite.txt` at line 60 as `LANGUAGE_C_ONLY call_bare_c` (added in Phase 01). The `LANGUAGE_C_ONLY` category also generates an Objective-C variant via the `setlanguage` macro. Test confirmed passing (4/4 units, 0% failure).

- [x] **Commit all changes to the `bare_function_fix` branch.** Stage the modified/new files:
  - `src/parser/srcMLParser.g` (grammar fix)
  - `test/parser/testsuite/call_bare_c.c.xml` (new test cases)
  - Any other files modified during the fix

  Create a commit with a descriptive message explaining what was fixed and why. Do NOT push to remote.
