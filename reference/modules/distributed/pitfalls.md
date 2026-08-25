# distributed Pitfalls

Module-specific pitfalls not covered by `reference/common/review-checklist.md`.

Only module-specific pitfalls go here — anything that applies to more than
one module belongs in the common layer.

## Capability Gates Beyond Device Type

Distributed tests gate on more than just "is a device available":

- **Multi-GPU / multi-process gates** (`TEST_MULTIGPU`, `TEST_SKIN_COLLECTIVES`,
  world-size and rank checks) express requirements a plain device swap cannot
  satisfy. Flag blanket conversions of these gates to `@onlyAccelerator` —
  an accelerator on a single-device machine still cannot run a 2-GPU test.
- **Backend gates** (NCCL/GLOO/UCC availability, `torch.cuda.nccl`) are
  Category C territory: collective backends are device-specific. Tests
  exercising backend internals are S3, not S2.

## Static-Review Bias

Distributed tests cannot be run locally during review (multi-process,
multi-GPU). Findings are static-analysis driven; prefer conservative
verdicts (flag for CI verification) over confident ones you cannot execute.

[Add more as they emerge from review practice.]
