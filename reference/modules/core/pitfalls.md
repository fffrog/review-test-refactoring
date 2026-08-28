# core Pitfalls

Module-specific pitfalls not covered by `reference/common/review-checklist.md`.

Only module-specific pitfalls go here — anything that applies to more than
one module belongs in the common layer.

## External References

Core test classes have three external-reference surfaces that break on
rename (see checklist 2a for the verification commands):

- **DecorateInfo in `common_methods_invocations.py`** — the dominant one for
  core op tests. `@ops`-generated tests are matched by exact class name in
  `is_active()`; a rename without updating entries silently drops skips/xfails.
- **CI exclude lists** — `.ci/pytorch/test_exclude_list.py` and
  `.ci/pytorch/*-trunk.yml` contain class names as strings; grep them when a
  class was renamed.
- **`.pytorch-disabled-tests.json`** — the disabled/flaky test mapping; a
  rename leaves stale entries that silently stop disabling the test.

## OpInfo Classes

`@ops`-parametrized classes are special among S1 classes: they still need
`instantiate_device_type_tests(..., only_for="cpu")` (not plain
`@instantiate_parametrized_tests`) because OpInfo decorators expect a
`device` argument, and their `hw_classification` is `CPU`, not `GENERIC`
(see checklist 3 and the standards' instantiation table).

Device-agnostic skips for `@ops` tests migrate to `@skipOps`
(`DecorateInfo`-based) rather than device-specific skip decorators — the
migration PRs covered `TestBwdGradients`, `TestFwdGradients`, and
`TestMeta`.

## Profiler Tests

`test/profiler/**` carries CUDA history that breaks other accelerators:

- **`ProfilerActivity.CUDA` on CPU-only profiling tests** makes other
  accelerators raise when they run the test — flag it for removal.
- **Hardcoded `torch.cuda.init` / CUDA-only profiler APIs** make the test
  S3: keep the original device scope (`only_for="cuda"`) instead of
  broadening to other accelerators.
- **Built-in event field names** (`self_cuda_time_total` and friends) are
  profiler terms — never rename them during device-agnostic refactoring.

[Add more as they emerge from review practice.]
