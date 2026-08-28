# graph Pitfalls

Module-specific pitfalls not covered by `reference/common/review-checklist.md`.

Only module-specific pitfalls go here — anything that applies to more than
one module belongs in the common layer.

## Sentinel Files (renames)

`test/dynamo_skips/`, `test/dynamo_expected_failures/`, and
`test/inductor_expected_failures/` hold sentinel files named after test
classes — **often 0 bytes**. They must be located by filename (`find -name`),
never by content (`grep`). When a device-parametrized class is renamed
(`TestFoo` → `TestFooDevice`), `instantiate_device_type_tests` renames the
device variants too (`TestFooCUDA` → `TestFooDeviceCUDA`), so sentinel files
named after the old variants must be renamed to match — otherwise expected
failures silently turn into CI failures. See checklist 2a.

## Inductor Device Abstractions

`GPU_TYPE` / `HAS_GPU` imported from `inductor_utils` are a leftover
inductor-specific device abstraction. In device-agnostic classes they must
be replaced with the standard `device` parameter machinery — flag them as
Major per checklist 7.

## Compile-Focused Bias

Many graph tests are legitimately S1 (tracing, graph transforms, code
checks) even though the module as a whole is compiler-centric. Do not
over-classify to S2 just because a file lives in `test/dynamo/**` or
`test/inductor/**` — classify each test on its own device usage.

## Stream APIs Under Dynamo Tracing

Do NOT apply the common unified-type rule to stream APIs inside
dynamo-traced test cases. Dynamo treats the device-module stream APIs
(`torch.cuda.stream`, `torch.cuda.Stream`, `torch.cuda.Event`) and the
unified types (`torch.Stream`, `torch.Event`) as distinct operations, so a
traced test keeps the original device-module calls — both forms need
separate testing. The unification only applies outside dynamo tracing.

## cpython Tests

`test/cpython/**` tests are CPython's own tests patched for dynamo.
`CPythonTestCase` carries the `GENERIC` tag at its base class, and the
directory is exempt from per-class tagging — do not flag missing
`hw_classification` attributes there.

[Add more as they emerge from review practice.]
