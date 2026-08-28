# Test Infrastructure APIs

Mechanisms in `torch/testing/_internal` that device-decoupled tests rely
on. Reviewers must know what each does and what to check when a test file
uses it. Shared by all modules.

## Hardware Classification Filtering

- `hw_classification` on a test class drives the `--hw-classification`
  runner flag: tests whose class tag does not match are skipped.
- Unclassified tests are skipped with a warning while the mechanism is
  experimental; function-based (module-level) tests are dropped under
  pytest because the tag is class-level.
- Checks: every class that defines `test_*` methods carries a tag,
  including intermediate and inherited classes; the tag matches the
  instantiation mechanism (see `test-classification-standards.md`).

## Instantiation Semantics

- `only_for` in `instantiate_device_type_tests` is an instantiation
  whitelist — whether a device variant actually runs still depends on that
  device's `is_available()`.
- Out-of-tree accelerators are instantiated by exporting
  `PYTORCH_TESTING_DEVICE_FOR_CUSTOM=<device>` before running tests.
- `bypass_device_restrictions` lets PrivateUse1-based backends run tests
  guarded by `@only*` decorators. A whitelist decorator therefore does not
  hard-block out-of-tree backends — do not treat `@onlyCUDA` as blocking
  OOT coverage by itself.

## Class-Level Exclusions

- `test_exclusions` / `set_test_configs()` on a device test class skips
  whole classes (`"*"`) or specific test methods (list of names), with an
  advanced form for dtype-aware exclusions.
- `op_allowlist` lets out-of-tree backends restrict which OpInfo tests run
  for them.
- Checks: exclusions default to `None`; unknown keys raise instead of
  being ignored; the exclusion config is validated (dead entries accumulate
  otherwise).

## Capability-Based Skips

- `@requires_capabilities(Capability.dtype.fp8, Capability.lib.triton, ...)`
  skips a test unless the class (via `DeviceTypeTestBase._capabilities`)
  declares those capabilities.
- Checks: capabilities come only from the predefined `Capability`
  namespace — undeclared requirements raise rather than silently skip; the
  capabilities lookup must not be cached per-class with `@lru_cache`
  (per-device subclasses such as `TestFooCUDA` overwrite the cache);
  capability facts per backend come from their own module
  (`common_cuda.py` etc.), not imported from other backends' packages.

## Distributed Backend Hook

- `DeviceTypeTestBase.distributed_backend()` returns the collective backend
  (NCCL/GLOO/UCC/...) for the device under test. Device-agnostic
  distributed tests use it instead of hardcoding a backend per device.
- Checks: backend selection goes through the hook or
  `dist.is_backend_available()`, not device-type conditionals.

## Rename Hazard: Disabled-Tests Mapping

- `.pytorch-disabled-tests.json` (repo root, `DISABLED_TESTS_FILE`) maps
  test names to disable reasons for flaky/unstable tests. Renaming a class
  silently stops disabling its tests. Check it on every rename (see
  checklist 2a).
