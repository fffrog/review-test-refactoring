# core Pitfalls

Module-specific pitfalls not covered by `reference/common/review-checklist.md`.

Only module-specific pitfalls go here — anything that applies to more than
one module belongs in the common layer.

## External References

Core test classes have two external-reference surfaces that break on rename
(see checklist 2a for the verification commands):

- **DecorateInfo in `common_methods_invocations.py`** — the dominant one for
  core op tests. `@ops`-generated tests are matched by exact class name in
  `is_active()`; a rename without updating entries silently drops skips/xfails.
- **CI exclude lists** — `.ci/pytorch/test_exclude_list.py` and
  `.ci/pytorch/*-trunk.yml` contain class names as strings; grep them when a
  class was renamed.

## OpInfo Classes

`@ops`-parametrized classes are special among S1 classes: they still need
`instantiate_device_type_tests(..., only_for="cpu")` (not plain
`@instantiate_parametrized_tests`) because OpInfo decorators expect a
`device` argument, and their `hw_classification` is `CPU`, not `GENERIC`
(see checklist 3 and the standards' instantiation table).

[Add more as they emerge from review practice.]
