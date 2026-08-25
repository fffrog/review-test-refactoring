---
name: test-refactoring
description: >
  Review PyTorch test files for correctness against cross-accelerator
  decoupling standards, routing each file through a module-specific review
  workflow (core, distributed, graph) backed by layered knowledge: shared
  common standards plus per-module rules. Review-only — this skill audits and
  reports findings, it never modifies test code. Use when asked to review,
  audit, or quality-check a PyTorch test file (test/** or torch/testing/**),
  to verify a test decoupling or refactoring change, to check S1/S2/S3
  classification correctness, or when the user opens a Python test file and
  asks for a quality check.
---

# Test Refactoring Review

Review PyTorch test files for correctness against cross-accelerator
decoupling standards. This skill is **review-only**: it audits and reports
findings, it never modifies test code. Because review rules differ per test
module, every file is routed through a module-specific review workflow backed
by **layered knowledge**: a common layer shared by all modules, plus a
per-module layer.

## Entry Workflow

Every request follows the same five steps:

1. **Identify the target.** Confirm the target is a PyTorch test file
   (`test/**` or `torch/testing/**`). Anything else is out of scope.

2. **Classify the file into a module.** This decides which review workflow
   and which module knowledge base apply. Authoritative rules:
   `reference/routing.md`.

   | Module | Detection | Knowledge |
   |--------|-----------|-----------|
   | `distributed` | path in `reference/modules/distributed/test-list.txt` | `reference/modules/distributed/` |
   | `graph` | path in `reference/modules/graph/test-list.txt` | `reference/modules/graph/` |
   | `core` | default — any test file not listed above | `reference/modules/core/` |

   - Exact membership in a `test-list.txt` is authoritative.
   - A path in multiple lists is a routing conflict — resolve per
     `reference/routing.md` (most specific module wins).
   - Unlisted files may be routed by path-pattern fallback, but confirm a
     non-default route with the user and record it in the list.
   - For a batch of files, classify each independently, then process module
     by module.

3. **Load the layered knowledge.**

   - **Common layer (always):** `reference/common/` — decoupling standards
     (S1/S2/S3), device API catalog, classification guide, device feature
     differences, review checklist.
   - **Module layer (per routed module):** `reference/modules/<module>/` —
     module scope (`README.md`), module review workflow (`workflow.md`),
     module pitfalls (`pitfalls.md`).

4. **Report the routing and knowledge selection.** Before starting the
   review, display what was routed and what was loaded, so the user can
   evaluate whether the decision is correct:

   ```text
   ## Routing & Knowledge Report
   - Target: <file path>
   - Module: <core | distributed | graph> (detection: <test-list.txt match | path-pattern fallback | default>)
   - Knowledge loaded:
     - reference/common/decoupling-standards.md
     - reference/common/device-api-catalog.yaml
     - reference/common/classification-guide.md
     - reference/common/device-features.md
     - reference/common/review-checklist.md
     - reference/modules/<module>/README.md
     - reference/modules/<module>/workflow.md
     - reference/modules/<module>/pitfalls.md
   - Workflow phases: assess, analyze, review, report
   ```

   This report is mandatory output for every run — it is how the routing and
   knowledge selection are evaluated for correctness. If the user corrects
   the routing, go back to step 2 with the corrected module.

5. **Follow the module review workflow.** Run the phases in the routed
   module's `workflow.md`: assess → analyze → review → report. The common
   layer supplies the standards every phase checks against; the module layer
   supplies module-specific rules and pitfalls.

## Knowledge Layers

| Layer | Location | Scope | Load when |
|-------|----------|-------|-----------|
| Common | `reference/common/` | S1/S2/S3 decoupling standards, API catalog, classification guide, device features, review checklist — applies to every module | Always |
| Module | `reference/modules/<module>/` | Scope, review workflow, and pitfalls specific to one module | After routing decides the module |

Layer rules:

- **Modules are parallel.** `core`, `distributed`, and `graph` are sibling
  modules. Each module's `workflow.md` is a complete, standalone review
  workflow — no module inherits phases from another. "core is the routing
  default" is a routing fact only; it implies nothing about knowledge
  hierarchy.
- **Never duplicate common knowledge in a module layer.** Module files state
  module-specific rules and pitfalls only; anything that applies to more
  than one module belongs in the common layer.
- **When in doubt, a rule belongs in common.** Move it into a module only
  when another module genuinely contradicts it.
- **Routing comes before knowledge.** If a file cannot be routed, stop and
  ask — do not guess a module (see `reference/routing.md`).

## Directory Layout

```
test-refactoring/
├── SKILL.md                       # this file — entry point and routing
└── reference/
    ├── routing.md                 # authoritative file -> module routing rules
    ├── common/                    # layer 1: shared knowledge, all modules
    │   ├── decoupling-standards.md
    │   ├── device-api-catalog.yaml
    │   ├── classification-guide.md
    │   ├── device-features.md
    │   └── review-checklist.md
    └── modules/                   # layer 2: per-module knowledge (parallel)
        ├── core/
        │   ├── README.md          # module scope and entry point
        │   ├── workflow.md        # core review workflow (complete, standalone)
        │   └── pitfalls.md
        ├── distributed/
        │   ├── README.md
        │   ├── workflow.md        # distributed review workflow (complete, standalone)
        │   ├── pitfalls.md
        │   └── test-list.txt      # module membership (authoritative)
        └── graph/
            ├── README.md
            ├── workflow.md        # graph review workflow (complete, standalone)
            ├── pitfalls.md
            └── test-list.txt      # module membership (authoritative)
```

## Content Status

- **Filled**: routing (`reference/routing.md`), the common layer
  (`reference/common/`), and all three module layers
  (`reference/modules/<module>/`) — migrated from the predecessor skills
  (`~/.claude/skills/review-test-refactoring/`,
  `~/.claude/skills/pytorch-test-refactoring/`).
- **Normalized during migration**: Category B semantics were conflicting in
  the sources (catalog header said Strategy 3, standards said Strategy 2).
  The filled content follows the standards: B is S2 in strategy, with
  call-site guidance. See `reference/common/classification-guide.md`.
- **Accumulating**: module-specific rules and pitfalls grow from review
  practice; each module file has an open section for them.

## Related

- Routing rules and how to add a module: `reference/routing.md`
- Review standards applied by every module: `reference/common/review-checklist.md`
