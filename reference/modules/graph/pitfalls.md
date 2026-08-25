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

[Add more as they emerge from review practice.]
