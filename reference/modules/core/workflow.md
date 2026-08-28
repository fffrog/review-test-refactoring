# core Review Workflow

The core module's review path. Complete and standalone: `distributed/` and
`graph/` have their own equally complete workflows, and no module inherits
phases from another. All modules share only the common layer.

## Phase 0: Assess

1. **Read the target per review mode.** Whole-file (`test/**`): read the
   entire file — you are auditing every class and test method, not a diff.
   Diff-based (`torch/testing/**`): read the diff and the enclosing
   class/function context.
2. **Inventory the file.** Note each test class, its instantiation mechanism
   (`TestCase`, `@instantiate_parametrized_tests`,
   `instantiate_device_type_tests` call and its `only_for`/`except_for` args),
   and its `hw_classification` tag. Count test methods per class.
3. **Run `lintrunner -a`.** Surface style, formatting, and import-order
   issues before the manual review. Note (do not fix) any failures.

## Phase 1: Analyze

1. Read `reference/common/test-classification-standards.md` — the S1/S2/S3
   classification hierarchy, decision tree, and strategy patterns.
2. Read `reference/common/api-classification-guide.md` — how to look up APIs in
   the catalog.
3. For every test, classify its **actual** device dependency:
   - Look up each device API the test uses in
     `reference/common/device-api-categories.yaml` (Category A/B/C).
   - Compare against the class it lives in. Mismatches are the raw material
     for Phase 2: false-CUDA in S3 classes (checklist 1a), Category C APIs
     in S2 classes (checklist 1b), device usage in S1 classes (checklist 1c).
   - Never classify from memory — ground every decision in the catalog.

## Phase 2: Review

Run the full checklist in `reference/common/review-checklist.md`, applying
every check (sections 1-9) to every class and test method in the file.

Then check this module's `pitfalls.md` for core-specific issues.

## Phase 3: Report

Report findings using the output format in
`reference/common/review-checklist.md`: Summary (with the routing facts
condensed into it), Findings, Verified Correct.

## Module-Specific Rules

Core files are top-level `test/test_*.py` (ops, nn, indexing, linalg, ...)
and `torch/testing/**` infrastructure. Rules specific to this module:

- **DecorateInfo is the dominant external reference.** Most core op tests
  are generated from `op_db` via `@ops`; renames break
  `DecorateInfo` entries in `common_methods_invocations.py` (checklist 2a).
  For `@ops` classes, the S1 mechanism is
  `instantiate_device_type_tests(only_for="cpu")` with
  `hw_classification = HardwareClassification.CPU` (checklist 3).
- **CI exclude lists** (`.ci/pytorch/test_exclude_list.py`, `*-trunk.yml`)
  reference core class names — grep them when a class was renamed.

[Add more rules as they emerge from review practice.]
