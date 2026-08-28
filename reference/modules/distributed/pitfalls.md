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

## Global Device Variables

No module-level device globals. A pattern like this fixes the device once
at import time, so every test runs on that device only and the file
bypasses the S1/S2/S3 machinery:

```python
device_type = (
    acc.type
    if (acc := torch.accelerator.current_accelerator(check_available=True))
    else "cpu"
)
```

Instead, each test takes a `device` parameter and the class is
instantiated with `instantiate_device_type_tests` (see
`reference/common/test-classification-standards.md`). The same applies to
any global that pins a device, backend, or world size.

## Backend Selection and Gating

- **Backend selection** in device-agnostic classes goes through
  `DeviceTypeTestBase.distributed_backend()` (or
  `dist.is_backend_available()`) — not device-type conditionals.
- **Skip messages must be accelerator-clear.** `TEST_SKIPS` entries and
  exit paths written for CUDA (`"no_cuda"`) read wrongly for other
  accelerators; the underlying checks should use
  `torch.accelerator.is_available()` and
  `torch.accelerator.device_count() < world_size`.
- **Enforce GPU-count requirements.** Multi-GPU tests must actually assert
  the required number of devices; a test that assumes N GPUs silently
  misbehaves on smaller machines.
- **Extract device-specific cases.** A device-agnostic distributed class
  must not carry backend-specific tests: move them into a dedicated
  device-specific class (with `@onlyAccelerator` or `only_for`) instead of
  leaving `if device_type == ...` branches.

## Static-Review Bias

Distributed tests cannot be run locally during review (multi-process,
multi-GPU). Findings are static-analysis driven; prefer conservative
verdicts (flag for CI verification) over confident ones you cannot execute.

[Add more as they emerge from review practice.]
