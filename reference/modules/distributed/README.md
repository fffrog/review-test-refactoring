# Module: distributed

## Scope

Distributed training tests: `test/distributed/**` — c10d, NCCL/GLOO/UCC
backends, process groups, collectives, FSDP, DDP, RPC, distributed
checkpoint, etc.

## Detection

Exact membership in `test-list.txt` (authoritative); path pattern
`test/distributed/**` as fallback for unlisted files (see
`reference/routing.md`).

## Knowledge

- Workflow: `workflow.md`
- Pitfalls: `pitfalls.md`
- Shared: `reference/common/`

## Module Characteristics

- **Multi-process launch**: most tests run via `MultiProcessTestCase` or
  `spawn`/`launch` helpers; review must account for rank/world-size logic.
- **Backend-specific tests**: NCCL/GLOO/UCC are Category C APIs, so S3 is
  more common here than in core.
- **Static-analysis-heavy review**: multi-process, multi-GPU tests cannot be
  run locally; findings rest on static analysis plus CI.

## Migration Sources

- `~/.claude/skills/pytorch-test-refactoring/reference/distributed/`
- Field-specific handling in `~/.claude/skills/pytorch-test-refactoring/flow.py`
