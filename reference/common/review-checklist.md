# Review Checklist

The review checklist applied by every module's review workflow
(`reference/modules/<module>/workflow.md`). Checks every class and test method
in a test file against `reference/common/test-classification-standards.md`.

Review modes are determined by the target's location (see SKILL.md):

- **Whole-file** (`test/**`, mandatory): apply every check below to every
  class and test method in the complete file — never just a diff.
- **Diff-based** (`torch/testing/**`, allowed when the input is a PR or
  diff): apply the checks to the changed hunks. Class-level checks (naming,
  instantiation mechanism, `hw_classification`, import cleanliness) are
  verified in the enclosing class/function context even in diff mode.

Every API classification decision must be grounded in
`reference/common/device-api-categories.yaml` (lookup procedure:
`reference/common/api-classification-guide.md`) — never rely on memory or
heuristics.

| Category | Description | Strategy Implication |
|----------|-------------|---------------------|
| **A** | APIs with `torch.accelerator` equivalents | **NOT device-specific** → Strategy 2 |
| **B** | General cross-backend concepts, no wrapper yet | **NOT device-specific** → Strategy 2 |
| **C** | Truly device-specific, no cross-device equivalent | **Strategy 3 only** |

**Rule**: Only Category C APIs justify Strategy 3. If a test uses only
Category A or B APIs, it must be Strategy 2 with `@onlyAccelerator`.

## Severity Levels

- **Blocker**: Test loss, wrong classification locking tests out of accelerators,
  broken instantiation (class won't run).
- **Major**: Wrong naming convention, wrong instantiation mechanism,
  stale imports that keep file classified as device_specific.
- **Minor**: Style issues, missed cleanup opportunities, suboptimal
  decorator ordering.

## 1. Classification Correctness

The single most impactful category of review finding. A wrong classification
either locks tests out of accelerators they could run on (Strategy 2
misclassified as Strategy 3) or causes test failures on accelerators that lack
the required features (Strategy 3 misclassified as Strategy 2).

### 1a. False-CUDA Detection (most common error)

For every test classified as Strategy 3 (`TestFooCUDA`) or using
`@onlyCUDA` / `device="cuda"`, ask:

> Is this test verifying a truly device-specific feature (Category C in the
> catalog), or is it just using CUDA as a device for generic computation?

**How to check**: Look up each `torch.cuda.*` API the test uses in
`reference/common/device-api-categories.yaml`. If every API it uses is Category A
or B, the test is misclassified — it should be Strategy 2 with
`@onlyAccelerator`.

**Red flags** (signals the test is wrongly classified as CUDA-specific):

| Code Pattern | What It Means | Severity |
|-------------|---------------|----------|
| Test with generic ops (add, softmax, matmul, loss) still has `@onlyCUDA` or `device="cuda"` | Should be Strategy 2 with `@onlyAccelerator` | Blocker |
| `.cuda()` / `.to("cuda")` used instead of `.to(device)` | Test hardcodes CUDA for no reason | Blocker |
| `torch.cuda.<api>` call where the catalog shows `torch.accelerator.<api>` exists | Category A — has cross-accelerator equivalent; replace with `torch.accelerator.*` | Major |
| `torch.cuda.Stream` / `torch.cuda.Event` used but test not marked as Strategy 3 | Category B — general concept; verify usage context, usually Strategy 2 | Info |
| `TEST_CUDA` import remains but no Strategy 3 CUDA tests exist in the file | Stale import keeps file classified as device_specific | Major |
| `SM80OrLater` (or similar capability constants) used in a `@skipCUDAIf` guard | Capability gate, not a device API — keep or generalize; not a false-CUDA signal | Info |

**Unnecessary `@onlyAccelerator`**: If `@onlyAccelerator` was ADDED to a test
that had no prior device restriction, verify the test genuinely requires an
accelerator. If it works on CPU, the restriction should have been removed
entirely.

### 1b. Over-Generalization Detection

Conversely, check that tests using Category C APIs were NOT incorrectly
generalized to Strategy 2. Consult `reference/common/device-api-categories.yaml`
→ `category_c` for the full per-backend lists. Key examples:

| Code Pattern | What It Means | Severity |
|-------------|---------------|----------|
| Test using any API from `category_c.cuda` in the catalog but placed in `TestFooDevice` with `@onlyAccelerator` | Will fail on non-CUDA accelerators | Blocker |
| Test using any API from `category_c.mps` in the catalog but placed in `TestFooDevice` | Will fail on non-MPS accelerators | Blocker |
| Test using any API from `category_c.xpu` in the catalog but placed in `TestFooDevice` | Will fail on non-XPU accelerators | Blocker |

**Dtype compatibility on MPS**: Even when a test uses only generic ops (no
Category C APIs), it may still fail on non-CUDA accelerators if it uses dtypes
not supported by that backend. The unsupported-dtype facts live in
`reference/common/backend-differences.md` (Dtype Support).

**How to validate**: For every test using `@onlyAccelerator` or the `device`
parameter, verify every dtype it exercises (from `@dtypes`, `_default_dtype`, or
inline tensor creation) against that table. Where a dtype is unsupported,
check that the test carries `@expectedFailureMPS`, `@dtypesIfMPS`, or
`@skipIfMPS` as appropriate.

**MPS coverage safety**: When MPS coverage is broadened (new `allow_mps=True`
or `@onlyAccelerator` replacing CUDA-only restriction), verify `@skipIfMPS` is
present unless MPS was already covered via `@dtypesIfMPS` or `@onlyMPS`.
Exception: `@skipIfMPS` is NOT required when the class is NOT instantiated for
MPS — MPS variants are only created when the class's
`instantiate_device_type_tests` call passes `allow_mps=True`. If no MPS variant
exists, the test cannot run on MPS and the skip is unnecessary.

**@onlyCPU to device-agnostic**: Verify each `@onlyCPU` test was individually
evaluated (not bulk-decided). Check that `device` param was added when
`@onlyCPU` was removed.

### 1c. Strategy 1 Correctness

For tests in a Strategy 1 class (`TestFoo` without device suffix):

| Check | What to Look For |
|-------|-----------------|
| No `device` parameter in method signature | `def test_foo(self)` not `def test_foo(self, device)` |
| No device-dependent decorators | No `@onlyCUDA`, `@onlyAccelerator`, `@skipCUDAIf`, etc. |
| No `.to(device)`, `.cuda()`, `device=device` in test body | All tensors are CPU |
| No cross-device tensor operations | Can't have CPU tensor op GPU tensor |

## 2. Naming Convention

Class renaming is **OPTIONAL**. The `hw_classification` member on TestCase
handles strategy classification; class names are no longer the primary
discriminator. The refactoring author decides whether to rename based on
external reference impact (see `reference/common/test-classification-standards.md` for
the decision framework).

**Do NOT flag a class name mismatch as an issue unless it is actively
misleading** (e.g., an S1 CPU-only class named `TestFooCUDA`). A class using
the original name with the correct strategy mechanism is valid.

For reference, the recommended naming convention (when the author chooses to
rename):

| Strategy | Recommended Name | Acceptable Alternative |
|----------|-----------------|----------------------|
| Strategy 1 (accelerator-unrelated) | `TestFoo` (original name) | Must NOT have device suffix |
| Strategy 2 (accelerator-agnostic) | `TestFooDevice` | Original name is fine |
| Strategy 3 (accelerator-specific) | Original name (`instantiate_device_type_tests` appends the device suffix) | Original name is fine |

### 2a. Cross-File Reference Integrity

**If the author kept the original class names, skip this check** — no external
references need updating. This is the primary benefit of not renaming.

**When a class IS renamed** (e.g., `TestIndexing` → `TestIndexingDevice`), the
old name may still be referenced in external configuration files. A rename
without updating these files causes CI breakage: dynamo expected-failure entries
stop matching, turning expected failures into unexpected failures.

**Files to check for stale class name references:**

| File/Directory | Example Reference | How to Verify |
|---------------|-------------------|---------------|
| `test/dynamo_skips/` | `TestIndexing.test_invalid_sparse_coo_values_cpu` | `find test/dynamo_skips/ -name "OldClassName*"` **(by filename, NOT grep — these are sentinel files, often 0 bytes)** |
| `test/dynamo_expected_failures/` | `TestIndexingCPU.test_byte_mask_cpu` | `find test/dynamo_expected_failures/ -name "OldClassName*"` **(by filename, NOT grep)** |
| `test/inductor_expected_failures/` | `TestIndexing.test_foo` | `find test/inductor_expected_failures/ -name "OldClassName*"` **(by filename, NOT grep)** |
| `torch/testing/_internal/common_methods_invocations.py` | `DecorateInfo(unittest.skip("..."), 'TestCommon', 'test_complex_half_reference_testing')` | Search for `'OldClassName'` string in `DecorateInfo(...)` constructor calls |
| `.ci/pytorch/test_exclude_list.py` | Test name in skip list | `grep -r "OldClassName\b" .ci/pytorch/` |
| `.ci/pytorch/*-trunk.yml` | Test name in CI config | `grep -r "OldClassName\b" .ci/` |
| `.pytorch-disabled-tests.json` (repo root) | Disabled/flaky test name mapping (`DISABLED_TESTS_FILE`) | `grep -n "OldClassName" .pytorch-disabled-tests.json` — a rename leaves stale entries that silently stop disabling the test |

**How to fix:** For each stale reference, update the class name to match the
new name. Verify which class actually owns each test — when a class is split
into multiple new classes (Strategy 1 + 2 + 3), tests may now live under
different class names (e.g., `TestIndexing.test_foo` might now be
`TestIndexingDevice.test_foo` or `TestIndexingCPU.test_foo`).

**For `common_methods_invocations.py` specifically:** `DecorateInfo` entries
use exact class name matching in `is_active()` — if `cls_name='TestCommon'`
but the test now lives in `TestCommonDevice`, the skip/xfail decorator is
silently dropped. To find broken entries:

```bash
python -c "
import torch
from torch.testing._internal.common_methods_invocations import op_db
from torch.testing._internal.opinfo.core import DecorateInfo

old_names = {'TestOldName1', 'TestOldName2'}  # fill in renamed classes
count = 0
for op in op_db:
    for d in op.decorators:
        if isinstance(d, DecorateInfo) and d.cls_name in old_names:
            print(f'{op.name}: cls_name={d.cls_name}, test_name={d.test_name}')
            count += 1
print(f'Total: {count} stale DecorateInfo entries')
"
```

Then replace the old class name string literal with the new one in
`torch/testing/_internal/common_methods_invocations.py`.

**Severity**: Blocker — broken DecorateInfo entries cause tests to silently
run when they should be skipped, or tests to be silently skipped when they
should run.

## 3. Instantiation Mechanism

Each class must use the mechanism its strategy prescribes. The full
mechanism-per-strategy mapping (including wrong mechanisms) is in
`reference/common/test-classification-standards.md` (Instantiation Mechanism
Comparison); the two critical checks:

**Critical**: Check that `instantiate_device_type_tests` is never used for
CPU-only (Strategy 1) classes — it creates useless per-device variants.

**Critical**: Check that no class uses both `instantiate_parametrized_tests`
and `instantiate_device_type_tests` — double instantiation causes test name
collisions.

## 4. API Replacement Correctness

For Strategy 2 tests, verify device-specific APIs were replaced with their
device-agnostic equivalents. **Consult
`reference/common/device-api-categories.yaml` → `category_a` for the authoritative
mapping.** The catalog defines every `torch.<device>.<api>` →
`torch.accelerator.<api>` replacement.

**Key checks:**

| Before (Wrong) | After (Correct) | Check |
|---------------|-----------------|-------|
| `@onlyCUDA` | `@onlyAccelerator` | Not left as `@onlyCUDA` |
| `@unittest.skipIf(not TEST_CUDA, ...)` | `@onlyAccelerator` | Not left as skip |
| `device="cuda"` | `device` parameter | No hardcoded `"cuda"` |
| `.cuda()` / `.to("cuda")` | `.to(device)` | No `.cuda()` calls |

For any `torch.cuda.<api>()` call remaining in a Strategy 2 test, check the
catalog: if Category A, it should be `torch.accelerator.<api>()`. If Category B,
use the unified type (e.g., `torch.Stream` instead of `torch.cuda.Stream`). If
Category C, the test belongs in Strategy 3.

**Return type compatibility**: Verify return type compatibility for all
`torch.accelerator.*` replacements, especially HIGH RISK APIs:
`current_device_index` (returns `int`, compare against `int`),
`set_device_index` (takes `int` arg), `get_device_capability` (return type
differs across backends). Consult `reference/common/device-api-categories.yaml`
type annotations.

**Remaining `torch.cuda` in S2 classes**: Scan each S2 class for remaining
`torch.cuda.*` calls — each must be either migrated to `torch.accelerator.*`
or the test moved to Strategy 3.

## 5. Import Cleanup

| Check | How to Verify |
|-------|---------------|
| `TEST_CUDA` import removed if no Strategy 3 CUDA tests remain | `grep "TEST_CUDA"` in the file |
| `TEST_MPS` import removed if no Strategy 3 MPS tests remain | `grep "TEST_MPS"` in the file |
| `TEST_XPU` import removed if no Strategy 3 XPU tests remain | `grep "TEST_XPU"` in the file |
| `@onlyCUDA` import removed if no Strategy 3 CUDA tests remain | `grep "onlyCUDA"` in the imports |
| `@onlyOn` import removed if all uses were replaced | `grep "onlyOn"` in the file |
| `@onlyNativeDeviceTypes` removal | `@onlyNativeDeviceTypes` / `@onlyNativeDeviceTypesAnd` are redundant on device-agnostic classes — REMOVE them. Before removing, verify dtype compatibility (`float64`/`complex128`/channels-last may be unsupported on MPS/MTIA) and add `@skipIfMPS`/`@dtypesIfMPS` if needed. |
| New imports are correct | `onlyAccelerator` from `common_device_type`, `torch.accelerator` if used |

## 6. Test Completeness

Verify every test method in the file is properly placed in an appropriate
strategy class. No test should be in a class that doesn't match its device
dependency level (e.g., a test using only CPU ops should not be in a
`TestFooCUDA` class).

| Check | How to Verify |
|-------|---------------|
| Every `def test_` belongs to the correct strategy class | Cross-reference each test's API usage against the catalog and its enclosing class name |
| Device instantiation present for Strategy 3 | `instantiate_device_type_tests(..., only_for="<device>")` with a `device` param on each method |
| No test logic unintentionally modified | Flag tests that appear incomplete or have empty bodies |
| No duplicate test bodies across device-specific classes | If identical test bodies appear across S3 classes, they belong in the S2 shared class |
| No device-specific artifacts in S2 classes | Scan for `_cuda` suffix in test method names, internal variable names like `cuda_out`, module-level helpers with `if device_type == "<backend>"` branches — clean these when the test is in an S2 class |
| No test lost in the change | Diff-based review: verify every original test method is accounted for (count `def test_` in old vs new), and each removed test re-appeared in the correct target class — deleting without migrating is the common failure mode. A test "lost" in refactoring is a regression. |

## 7. Common Pitfalls

| Pitfall | Detection | Severity |
|---------|-----------|----------|
| `@onlyAccelerator` used as class decorator | `@onlyAccelerator\nclass TestFoo` — breaks `instantiate_device_type_tests` | Blocker |
| Strategy 1 class has `device` parameter | `def test_foo(self, device)` in `TestFoo` (no Device suffix) | Blocker |
| `skipIfXpu`/`skipIfCUDA` from `common_utils` in device-agnostic class | These skip ALL variants, not just the target device | Major |
| `GPU_TYPE`/`HAS_GPU` from `inductor_utils` not converted | Leftover inductor-specific device abstraction | Major |
| Mixed `device` param and hardcoded `"cuda"` in same class | Inconsistent; some tests use device param, others hardcode | Major |
| `instantiate_device_type_tests` call references wrong class name | Class name in `globals()` call doesn't match actual test class — class never instantiated | Blocker |
| `except_for`/`only_for`/`allow_mps`/`allow_xpu` args missing from instantiation | Device allowlists not applied to current `instantiate_device_type_tests` call | Major |
| Category A/B API treated as if it makes a test CUDA-specific | Test locked to CUDA unnecessarily; check the catalog | Major |
| Missing blacklist skip decorators | `@skipXPU`, `@skipMPS`, `@skipMeta` absent — these document known gaps. If the original file had them and they're now gone, that's a regression | Blocker |
| `@onlyAccelerator` used without dtype compatibility check | Test runs on MPS/XPU but uses `complex128` or `float64` (unsupported on MPS). For every test using `@onlyAccelerator`, verify every dtype the test uses is supported on ALL target backends. If not, add `@expectedFailureMPS`, `@dtypesIfMPS`, or a skip decorator. | Blocker |
| Test class name doesn't match OpInfo DecorateInfo references | `DecorateInfo` entries in `common_methods_invocations.py` use exact class name matching in `is_active()`. If the test class was RENAMED, verify DecorateInfo entries were updated (section 2a). If the author kept the original name, this check is a no-op. | Blocker |
| **Flagging an original class name as "wrong" when the author chose not to rename** | Renaming is optional. If the author kept the original name (e.g., `TestFoo` for an S2 class), do NOT flag it unless the name is actively misleading (e.g., a CPU-only class named `TestFooCUDA`). The `hw_classification` member handles classification. | N/A — reviewer guidance |
| `SM80OrLater` (or similar capability constants) flagged for removal or silently deleted | Capability gates are kept as targeted skips (`@skipCUDAIf(not SM80OrLater, ...)`) or generalized — never silently dropped, and they do not make a test S3 (standards: CUDA Capability Constants) | N/A — reviewer guidance |
| `@unittest.skipIf(not TEST_CUDA, ...)` leftover in S2 class | Should be `@onlyAccelerator` | Major |
| `@skipIfMPS`/`@skipXPU`/`@skipCUDAIf` applied to method without `device` parameter | These decorators check the `device` kwarg and silently fail if missing | Blocker |
| Missing `hw_classification` class attribute | Every test class must have `hw_classification = HardwareClassification.XXX`. Missing attr causes test runner to skip or misroute tests. | Blocker |
| Incorrect `hw_classification` value | Value must match the class mechanism: GENERIC for S1, ACCELERATOR for S2, CUDA/MPS/XPU for S3 per device, CPU for S1-with-`@ops`. A wrong value (e.g., GENERIC on an S2 class) breaks `--hw-classification` filtering. | Blocker |
| `HardwareClassification` not imported | Must be imported from `torch.testing._internal.common_utils` and merged alphabetically into the existing import block. | Blocker |
| `instantiate_device_type_tests` call changes the original device scope (dropped `only_for`, added `allow_xpu`/`allow_mps`/`except_for` without evidence) | Refactoring must preserve before/after behavior — keep the original device set unless the tests were verified on the new devices (standards: Refactoring Invariants) | Major |
| Module-level `only_for` variable instead of a literal in the instantiate call | Inline the device list in the call; prefer `except_for` when excluding a minority of devices (standards: Refactoring Invariants) | Minor |
| Backend capability gap marked with xfail instead of skip | Unsupported dtype on a backend is a known gap — use `@dtypesIf<Device>` / `@skip<Device>If`, not xfail (standards: Refactoring Invariants) | Minor |
| S2 class restricted to accelerators at class level while the tests would run on CPU | A `TestFooDevice` class includes CPU; `@onlyAccelerator` is per-test, only for tests that cannot run on CPU (standards: Refactoring Invariants) | Major |
| `DeviceType[dev]`-style lookup to map a device string to a type | Breaks for PrivateUse1-based backends (`DeviceType.<registered_name>` vs `DeviceType.PrivateUse1`). Compare `str(device_type)` values instead | Major |
| Multiple cases executed in a plain loop inside one test | One failure hides the rest — convert to `@parametrize` or `subTest` when touching the test | Minor |
| Platform-helper booleans (`flex_attention_supported_platform` and similar) used to skip instead of explicit per-device skips | Helpers silently skip PrivateUse1-based backends they do not know and hide which devices are skipped — prefer explicit `@skip<Device>If` conditions (standards: Refactoring Invariants) | Major |

## 8. Decorator Ordering

For Strategy 2 tests, decorators must be ordered correctly. The `device`
parameter is filled in by `instantiate_device_type_tests`, and other
parametrization decorators fill additional arguments:

```python
# Correct: @dtypes closest to method, @onlyAccelerator above
@onlyAccelerator         # outermost (skip if CPU)
@dtypes(torch.float32)   # parametrization
def test_foo(self, device, dtype):
    ...

# Wrong — @onlyAccelerator below @dtypes may cause issues
@dtypes(torch.float32)
@onlyAccelerator
def test_foo(self, device, dtype):  # incorrect
    ...
```

## 9. HardwareClassification Tag

Every test class must have a `hw_classification` class attribute matching its
strategy and instantiation mechanism (see the mapping in
`reference/common/test-classification-standards.md`). **This is mandatory** — the test
runner uses `--hw-classification` to filter test execution by hardware
category.

**How to verify:**

| Check | How to Verify |
|-------|---------------|
| Import present | `grep "HardwareClassification" <file>` — must be imported from `torch.testing._internal.common_utils` |
| Every class tagged | `grep "hw_classification" <file>` — count must equal number of TestCase subclasses, including intermediate and inherited classes that define `test_*` methods |
| Value matches strategy | Cross-reference each class's mechanism (instantiate call + device params) against the table in the standards |
| Import merged alphabetically | `HardwareClassification` must appear in the existing `common_utils` import block in alphabetical order |

**Severity**: Blocker — missing or incorrect `hw_classification` causes the
test runner to skip or misroute tests.

## 10. PR Hygiene (diff-based reviews)

When the input is a PR or diff, also report process-level findings (all
Minor):

- **Oversized refactor.** A single PR touching many classes is hard to
  review and slow to merge. Suggest splitting — additions and deletions as
  separate commits, or one file per PR.
- **Mixed unrelated changes.** Format-only edits, pytest parametrize
  rewrites, or other files bundled into a decoupling PR. Suggest splitting
  them out so the refactor itself stays reviewable.
- **Unused leftover code.** Helpers, variables, or decorators that became
  dead after the refactor (`@onlyCUDA` imports, module-level device
  globals, unused functions).

## Review Output Format

Structure your review as follows:

```
## Review: <test file path>

### Summary
- Target: <file path | PR ref>
- Mode: whole-file / diff-based
- Module: <core | distributed | graph> (detection: <directory match | test-list.txt match | default>)
- Knowledge loaded: reference/common/ (6 files) + reference/modules/<module>/{README.md, workflow.md, pitfalls.md}
- Workflow phases: assess, analyze, review, report
- File(s) reviewed: 1
- Classification: correct / N issues found
- Naming: correct / N issues found
- API replacements: correct / N issues found
- Completeness: all tests properly placed / N issues found

### Findings

- **Blockers (must fix)**
    - [ ] **<file:line>**: <issue description>
        - Fix: <suggested fix>
- **Major (should fix)**
    - [ ] **<file:line>**: <issue description>
- **Minor (nice to have)**
    - [ ] **<file:line>**: <issue description>

### Verified Correct
- <list of things that are correct per the standards>
```

The routing facts (module, detection, knowledge loaded, workflow phases)
are condensed into the Summary so the final report stays self-contained.
The full Routing & Knowledge Report is emitted once at the start of the
run (SKILL.md step 4) and is not repeated here.
