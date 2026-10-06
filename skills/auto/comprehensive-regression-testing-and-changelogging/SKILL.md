---
name: comprehensive-regression-testing-and-changelogging
description: Use when fixing bugs or updating codebases that require regression tests and changelog entries.
---
<body>
1. Identify all bugs or issues fixed across the codebase and ensure every public function has complete type annotations on all parameters and return values.
2. Create or update a dedicated test file (e.g., `tests/test_regressions.py`) containing at least one test function per bug fixed, ensuring all tests pass successfully under the test runner.
3. Record each fix in the project changelog (e.g., `CHANGELOG.md`) under the designated heading (such as `## Unreleased`) as a clear bullet point following the required format (`- fix(<function name>): <short description>`).
4. Run the full test suite and verify that all test files remain intact and passing.
5. Self-Check: Are all public functions fully type-annotated? Are there dedicated regression tests for every bug? Are all fixes documented in the changelog?
