# Module: graph

## Scope

Graph-mode / compiler tests: `test/jit/**`, `test/fx/**`, `test/dynamo/**`,
`test/inductor/**`, `test/export/**`, `test/onnx/**`, `test/functorch/**`,
`test/higher_order_ops/**`, `test/lazy/**`, `test/cpp/**`,
`test/cpython/**` and related top-level `test/test_*_jit*.py` files.

Note: this bucket is deliberately broad for now. If a sub-area grows its own
workflow or rules, split it into its own module (see "Adding a Module" in
`reference/routing.md`).

## Detection

Directory routing: everything under the graph directories listed above
belongs to this module (authoritative — see `reference/routing.md`).
`test-list.txt` additionally holds top-level files (`test/test_*jit*.py`,
`test/test_fx*.py`, ...) outside those directories.

## Knowledge

- Workflow: `workflow.md`
- Pitfalls: `pitfalls.md`
- Shared: `reference/common/`

## Module Characteristics

- **Compile-focused tests**: much of the content is tracing, graph
  transforms, and code checks — legitimately S1 more often than in core.
- **Sentinel files**: dynamo/inductor skip and expected-failure sentinel
  files are the dominant external-reference surface on rename.
- **Inductor device abstractions**: `GPU_TYPE`/`HAS_GPU` leftovers appear
  almost exclusively here.

## Migration Sources

- `~/.claude/skills/pytorch-test-refactoring/reference/graph/`
- Field-specific handling in `~/.claude/skills/pytorch-test-refactoring/flow.py`
