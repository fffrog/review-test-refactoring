# distributed Review Workflow

The distributed module's review path. Complete and standalone: `core/` and
`graph/` have their own equally complete workflows, and no module inherits
phases from another. All modules share only the common layer.

## Phase 0: Assess

1. **Read the target per review mode.** Whole-file (`test/**`): read the
   entire file — you are auditing every class and test method, not a diff.
   Diff-based (`torch/testing/**`): read the diff and the enclosing
   class/function context.
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

Report findings using the output format in
`reference/common/review-checklist.md`: the final report opens with the
Routing & Knowledge Report (repeated there so the final output is
self-contained), then Summary, Findings, Verified Correct.

## Module-Specific Rules

- **Assume standard backend setup; do not flag corner cases.** Backends
  dynamically register into `dist.Backend.default_device_backend_map` at
  load time, so direct lookups like `default_device_backend_map[acc.type]`
  are the standard pattern and not a finding. Review against the standard,
  registered path — hypothetical unregistered-backend failures are out of
  scope.

## Review Constraints

- Distributed tests are hard to run locally during review (multi-process,
  multi-GPU). Verification relies on static analysis plus CI results; do not
  promise local reproduction.
