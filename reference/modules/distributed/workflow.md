# distributed Review Workflow

The distributed module's review path. Complete and standalone: `core/` and
`graph/` have their own equally complete workflows, and no module inherits
phases from another. All modules share only the common layer.

## Phase 0: Assess

1. **Read the entire test file.** You are auditing every class and test
   method, not a diff.
2. **Inventory the file.** Note each test class, its instantiation mechanism,
   `hw_classification` tag, and — specific to this module — how the test
   launches: `MultiProcessTestCase`, `spawn`/`launch` helpers, subprocess
   rank logic, and backend selection (NCCL/GLOO/UCC).
3. **Run `lintrunner -a`.** Note (do not fix) any failures.

## Phase 1: Analyze

1. Read `reference/common/test-classification-standards.md` — the S1/S2/S3
   classification hierarchy, decision tree, and strategy patterns.
2. Read `reference/common/api-classification-guide.md` — how to look up APIs in
   the catalog.
3. For every test, classify its **actual** device dependency:
   - Collective/backend APIs (NCCL, GLOO, UCC, process-group internals) are
     Category C in `reference/common/device-api-categories.yaml` → those tests
     are S3 by default. This makes S3 more common here than in core.
   - Multi-process and multi-GPU requirements (rank count, world size) are
     capability gates beyond device type — treat them separately from the
     S1/S2/S3 classification.
   - Never classify from memory — ground every decision in the catalog.

## Phase 2: Review

Run the full checklist in `reference/common/review-checklist.md`, applying
every check (sections 1-9) to every class and test method in the file.

Then check this module's `pitfalls.md` for distributed-specific issues.

## Phase 3: Report

Report findings grouped by severity (Blocker / Major / Minor) using the
output format in `reference/common/review-checklist.md`. The routing report
(module, detection method, loaded knowledge files) was already emitted
before Phase 0 — keep the review itself in the standard format.

## Module-Specific Rules

[Add rules specific to `test/distributed/**` files as they emerge from
review practice.]

## Review Constraints

- Distributed tests are hard to run locally during review (multi-process,
  multi-GPU). Verification relies on static analysis plus CI results; do not
  promise local reproduction.
