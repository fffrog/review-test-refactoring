# graph Review Workflow

The graph module's review path. Complete and standalone: `core/` and
`distributed/` have their own equally complete workflows, and no module
inherits phases from another. All modules share only the common layer.

## Phase 0: Assess

1. **Read the entire test file.** You are auditing every class and test
   method, not a diff.
2. **Inventory the file.** Note each test class, its instantiation mechanism,
   and `hw_classification` tag. Graph files often mix plain `TestCase`
   classes (compile-focused, CPU) with device-parametrized classes — note
   which is which up front.
3. **Run `lintrunner -a`.** Note (do not fix) any failures.

## Phase 1: Analyze

1. Read `reference/common/decoupling-standards.md` — the S1/S2/S3
   classification hierarchy, decision tree, and strategy patterns.
2. Read `reference/common/classification-guide.md` — how to look up APIs in
   the catalog.
3. For every test, classify its **actual** device dependency:
   - Many graph tests are compile-focused (tracing, graph transforms, code
     checks) and legitimately S1 — do not push them to S2 just because the
     file touches dynamo/inductor.
   - For device-parametrized classes, apply the same catalog-grounded
     classification as core.
   - Never classify from memory — ground every decision in the catalog.

## Phase 2: Review

Run the full checklist in `reference/common/review-checklist.md`, applying
every check (sections 1-9) to every class and test method in the file.

Then check this module's `pitfalls.md` for graph-specific issues — in
particular the sentinel-file rename checks, which matter most in this
module.

## Phase 3: Report

Report findings grouped by severity (Blocker / Major / Minor) using the
output format in `reference/common/review-checklist.md`. The routing report
(module, detection method, loaded knowledge files) was already emitted
before Phase 0 — keep the review itself in the standard format.

## Module-Specific Rules

[Add rules specific to graph/compiler test files as they emerge from review
practice.]
