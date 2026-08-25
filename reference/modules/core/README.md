# Module: core

## Scope

Default module: every PyTorch test file not claimed by `distributed` or
`graph` — top-level `test/test_*.py` (ops, nn, indexing, linalg, ...) and
`torch/testing/**` infrastructure.

## Detection

No `test-list.txt` — `core` is the routing default (see
`reference/routing.md`).

## Knowledge

- Workflow: `workflow.md` (complete, standalone review workflow)
- Pitfalls: `pitfalls.md`
- Shared: `reference/common/`

core is one of three parallel modules — no module is the baseline for
another (see `reference/routing.md`).

## Migration Sources

- `~/.claude/skills/pytorch-test-refactoring/SKILL.md`
- `~/.claude/skills/pytorch-test-refactoring/agent/skills/refactor-test-decoupling/SKILL.md`
- `~/.claude/skills/pytorch-test-refactoring/reference/` (core uses the legacy
  root reference dir)
- `~/.claude/skills/review-test-refactoring/SKILL.md`
