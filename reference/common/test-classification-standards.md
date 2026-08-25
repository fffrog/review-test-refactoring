# Test Classification Standards

The normative standard for how PyTorch tests are classified into S1/S2/S3 and
what a correctly decoupled test file looks like. The review checklist
(`reference/common/review-checklist.md`) audits test files against this file.
Applies to every module; module-specific rules live in
`reference/modules/<module>/`.

**Boundary:**
- API-level classification data: `reference/common/device-api-categories.yaml`
- API lookup procedure and A/B/C semantics: `reference/common/api-classification-guide.md`
- Per-backend feature facts: `reference/common/backend-differences.md`

## Classification

Every test falls into one of three strategies. Classification is hierarchical:
**S3 > S2 > S1** — a test using ANY Category C API is S3.

| Strategy | Definition | Mechanism |
|----------|-----------|-----------|
| **Accelerator-unrelated (S1)** | No device usage; CPU only | `instantiate_parametrized_tests()` or plain `TestCase` |
| **Accelerator-agnostic (S2)** | Uses a device but only generic accelerator APIs | `instantiate_device_type_tests()` |
| **Accelerator-specific (S3)** | Requires a particular accelerator's unique features | `instantiate_device_type_tests(only_for="<device>")` |

### Device API Categories

| Category | Description | Strategy |
|----------|-------------|----------|
| **A** — has `torch.accelerator` equivalent | `empty_cache`, `synchronize`, `CUDAGraph`, `memory_allocated`, `current_device` | S2 |
| **B** — general concept, no accelerator wrapper | `Stream`, `Event`, `manual_seed`, `get_device_properties` | S2 |
| **C** — truly device-specific | NCCL, NVTX, cuDNN, GDS, Jiterator, Metal shaders, SYCL handles | S3 |

Full semantics — call-site actions per category and the Category B
reconciliation — live in `reference/common/api-classification-guide.md`.

**Only Category C makes a test S3.** If you can replace `"cuda"` with `"mps"` or
`"xpu"` and the test still makes logical sense, it's S2.

### Decision Tree

```
Does the test reference a device?
├─ NO → S1
├─ YES → What device APIs?
│  ├─ Generic only (torch.device(device), make_tensor(..., device=device)) → S2
│  ├─ Category A or B APIs → S2
│  ├─ Category C APIs → S3
│  └─ Hard to tell → Leave as-is

What decorators?
├─ Blacklist (@skipXPU, @skipCUDAIf, @skipMPS, @skipMeta) → KEEP
├─ Whitelist (@onlyCUDA, @onlyOn, @unittest.skipIf(not TEST_CUDA, ...)) → ENLARGE to @onlyAccelerator

> **Note:** An `if device_type == "<backend>"` conditional in the test body does
> NOT make a test S3 — only Category C API calls do.
```

## Blacklist vs. Whitelist Decorators

| Decorator Type | Examples | Principle | Action |
|---------------|----------|-----------|--------|
| **Blacklist** (explicit skips) | `@skipXPU`, `@skipCUDAIf`, `@skipMPS`, `@skipMeta` | Documents a **known gap** — intentional and informed | **KEEP as-is** |
| **Whitelist** (restrictive) | `@onlyCUDA`, `@onlyOn(["cuda","xpu"])`, `@unittest.skipIf(not TEST_CUDA, ...)` | Artificially **restricts** — usually historical accident | **ENLARGE** to `@onlyAccelerator` |
| **Whitelist** (restrictive) | `@onlyCPU` | Artificially **restricts** | **REMOVE** — make test device-agnostic (add `device` param). MUST evaluate each `@onlyCPU` test individually. Default to S2 unless the test genuinely tests CPU-only dispatch behavior. |

**`@onlyNativeDeviceTypes` / `@onlyNativeDeviceTypesAnd`** are redundant on
device-agnostic classes — device instantiation already scopes to the right
devices. REMOVE them.

## CUDA Capability Constants

Constants like `SM80OrLater` (compute-capability gates from
`torch.testing._internal.common_cuda`) are capability checks, not
device-specific APIs. They do **not** make a test S3, and removing them
wholesale is **not** required. Two valid treatments:

1. **Generalize to a capability.** Where the check can be expressed with a
   generic equivalent (e.g. a `torch.accelerator` capability query), convert
   it and keep the test in an S2 class.
2. **Keep it as a targeted skip.** Leave the constant in place and use it
   in a device-type-aware skip so only the CUDA variants lacking the
   capability are skipped — the test stays S2 and non-CUDA accelerators are
   unaffected:

   ```python
   @onlyAccelerator
   @skipCUDAIf(not SM80OrLater, "requires SM80+")
   def test_foo(self, device):
       ...
   ```

Never silently delete the check — the test would then run on hardware that
cannot support it. And never demote the whole test to S3 just because a
capability constant appears in it.

## False-CUDA Patterns (→ S2, NOT S3)

These almost always indicate S2:

| Pattern | Why Not CUDA-Specific | Action |
|---------|----------------------|--------|
| `@onlyCUDA` on standard ops (add, softmax, matmul) | The op works on any accelerator | `@onlyAccelerator` + `device` param |
| `.cuda()` / `.to("cuda")` on tensors | Just device placement | `.to(device)` |
| `device="cuda"` in tensor creation | Any device would work | `device` param |
| `@unittest.skipIf(not TEST_CUDA, ...)` | Proxy for "needs accelerator" | `@onlyAccelerator` |
| Test name contains `_cuda` | Naming, not functional | Remove suffix |

**Caveats:**
- Do NOT enlarge `@onlyCUDA` → `@onlyAccelerator` if the test had no prior device restriction — remove the restriction entirely instead.
- Keep `@onlyCUDA` if the test relies on backend-specific behavioral guarantees (NaN handling, determinism, precision, rounding modes).

## Naming Convention

Class renaming is **OPTIONAL**. The `hw_classification` member on TestCase
handles strategy classification; class names are no longer the primary
classification mechanism. Decide whether to rename based on external reference
impact.

### Recommended Names (when renaming)

| Strategy | Recommended Class Name | Instantiation | Example |
|----------|----------------------|---------------|---------|
| **S1** | `TestFoo` (keep original name) | `@instantiate_parametrized_tests` or plain `TestCase` | `TestBinaryUfuncs` |
| **S2** | `TestFooDevice` | `instantiate_device_type_tests()` | `TestBinaryUfuncsDevice` |
| **S3** | `TestFoo` (original name — `instantiate_device_type_tests` appends the device) | `instantiate_device_type_tests(only_for="<device>")` | `TestBinaryUfuncsCUDA` |

### Renaming Decision

**When to rename:**
- The original class name has few or no external references (DecorateInfo entries, `dynamo_skips/`, `dynamo_expected_failures/`)
- The rename improves clarity (e.g., `TestFoo` → `TestFooDevice` makes the strategy obvious)

**When to keep the original name:**
- The class has many external references that would need updating
- Renaming would risk silently breaking CI (stale DecorateInfo, dynamo skip files)
- The original name is already clear enough

**How to decide:**
1. Check for DecorateInfo references: `grep "cls_name.*OldName" torch/testing/_internal/common_methods_invocations.py`
2. Check for dynamo skip/expected-failure files: `find test/dynamo_skips/ test/dynamo_expected_failures/ -name "OldName*"`
3. If zero or very few external refs → rename is safe
4. If many external refs → keep the original name (avoids breaking cross-file references)

`instantiate_device_type_tests` **removes** the generic class from scope and
replaces it with per-device variants (`TestFooDeviceCPU`, `TestFooDeviceCUDA`,
etc.). `instantiate_parametrized_tests` keeps the class discoverable.

**S3 mechanism**: S3 classes use `instantiate_device_type_tests(<Class>, globals(), only_for="<device>")` with a `device` parameter on each test method — they do NOT use a plain `TestCase` with a `setUp` guard.

## Strategy Patterns

### Strategy 1 (S1) — Accelerator-Unrelated

Zero device dependency. CPU tensors only, no `device` parameter. Two patterns:

**Pattern A — plain `TestCase`** (no parametrization):
```python
from torch.testing._internal.common_utils import HardwareClassification

class TestFoo(TestCase):
    hw_classification = HardwareClassification.GENERIC

    def test_basic_addition(self):
        a = torch.randn(3, 3)
        b = torch.randn(3, 3)
        self.assertEqual(a + b, torch.add(a, b))
```

**Pattern B — `@instantiate_parametrized_tests`** (has `@parametrize`/`@ops`/`@dtypes`):
```python
from torch.testing._internal.common_utils import HardwareClassification

@instantiate_parametrized_tests
class TestFoo(TestCase):
    hw_classification = HardwareClassification.GENERIC

    @parametrize("dtype", [torch.float32, torch.float64])
    def test_dtype_behavior(self, dtype):
        t = torch.randn(3, 3, dtype=dtype)
        self.assertEqual(t.softmax(0).sum(0), torch.ones(3, dtype=dtype))
```

**Why not `instantiate_device_type_tests`?** It creates per-device variants
(`TestFooCPU`, `TestFooCUDA`, …) — wasteful when all variants do the same
CPU-only work.

### Strategy 2 (S2) — Accelerator-Agnostic

Tests that use a `device` parameter but only need generic accelerator APIs. This
is the highest-impact refactoring — it unlocks tests for all accelerators at
once.

**Before** (false-CUDA):
```python
from torch.testing._internal.common_cuda import TEST_CUDA

class TestFoo(TestCase):
    @unittest.skipIf(not TEST_CUDA, "no CUDA")
    def test_softmax_cuda(self):
        t = torch.randn(3, 3, device="cuda")
        result = t.softmax(0)
        self.assertEqual(result.sum(0), torch.ones(3, device="cuda"))

    @onlyCUDA
    @skipXPU  # XPU doesn't support this op yet
    def test_matmul_cuda(self, device):
        a = torch.randn(3, 3, device=device)
        b = torch.randn(3, 3, device=device)
        self.assertEqual(a @ b, torch.matmul(a, b))
```

**After** (accelerator-agnostic):
```python
from torch.testing._internal.common_device_type import (
    instantiate_device_type_tests, onlyAccelerator,
)
from torch.testing._internal.common_utils import HardwareClassification

class TestFooDevice(TestCase):
    hw_classification = HardwareClassification.ACCELERATOR

    @onlyAccelerator
    def test_softmax(self, device):
        t = torch.randn(3, 3, device=device)
        result = t.softmax(0)
        self.assertEqual(result.sum(0), torch.ones(3, device=device))

    @onlyAccelerator
    @skipXPU  # Still here — known gap
    def test_matmul(self, device):
        a = torch.randn(3, 3, device=device)
        b = torch.randn(3, 3, device=device)
        self.assertEqual(a @ b, torch.matmul(a, b))

instantiate_device_type_tests(TestFooDevice, globals())
```

**Key rules:**
- `@onlyAccelerator` is a **method** decorator, NOT a class decorator — on a class it replaces the class with a function and `instantiate_device_type_tests` fails.
- Use device-type-aware skips in S2 classes: `skipXPUIf(True, msg)` / `skipCUDAIf(condition, msg)` from `common_device_type` (not `common_utils`) — these check `self.device_type` and only skip the specific device variant.
- Category A APIs (`empty_cache`, `synchronize`, `CUDAGraph`, `memory_*`) have `torch.accelerator.*` equivalents — they do NOT make a test CUDA-specific.
- Category B APIs (`Stream`, `Event`) are general concepts — they do NOT make a test CUDA-specific.

### Strategy 3 (S3) — Accelerator-Specific

Tests requiring a particular accelerator's unique (Category C) features.

```python
from torch.testing._internal.common_device_type import instantiate_device_type_tests
from torch.testing._internal.common_utils import HardwareClassification

class TestFoo(TestCase):
    hw_classification = HardwareClassification.CUDA

    def test_cuda_stream(self, device):
        s = torch.cuda.Stream()
        ...

instantiate_device_type_tests(TestFoo, globals(), only_for="cuda")
```

**Key rules:**
- Use `instantiate_device_type_tests(only_for="<device>")`; every test method takes a `device` parameter. Do NOT use a plain `TestCase` with a `setUp` guard.
- Keep the original class name — `instantiate_device_type_tests` appends the device name (`TestFoo` → `TestFooCUDA`). Do NOT pre-suffix (`TestFooOnCUDA` + `only_for="cuda"` → `TestFooOnCUDACUDA`).

## Instantiation Mechanism Comparison

| Mechanism | Creates Device Variants? | Generic Class Discoverable? | hw_classification | Use When | Wrong Mechanism |
|-----------|--------------------------|----------------------------|-------------------|----------|-----------------|
| Plain `TestCase` | No | Yes | `GENERIC` | No parametrization needed | `instantiate_device_type_tests` (useless per-device variants) |
| `instantiate_parametrized_tests()` | No | Yes | `GENERIC` | Tests with `@parametrize`/`@ops`/`@dtypes`, no device dependency | `instantiate_device_type_tests` |
| `instantiate_device_type_tests(only_for="cpu")` | Yes (CPU only) | No (removed from scope) | `CPU` | S1 classes with `@ops` (OpInfo tests need device instantiation) | `@instantiate_parametrized_tests` |
| `instantiate_device_type_tests()` | Yes (CPU, CUDA, MPS, …) | No (removed from scope) | `ACCELERATOR` | Tests with a `device` parameter, works on any accelerator | `@instantiate_parametrized_tests` |
| `instantiate_device_type_tests(only_for="<device>")` | Yes (single device) | No (removed from scope) | `CUDA` / `MPS` / `XPU` | S3 classes — Category C APIs | Plain `TestCase` with `setUp` guard, or `@instantiate_parametrized_tests` |

## HardwareClassification Tags

Every test class must carry a `hw_classification` class attribute matching its
strategy and instantiation mechanism. The test runner uses
`--hw-classification` to filter test execution by hardware category.

| Class Mechanism | Expected `hw_classification` |
|----------------|------------------------------|
| Plain `TestCase` or `@instantiate_parametrized_tests`, no `device` param | `HardwareClassification.GENERIC` |
| `instantiate_device_type_tests(only_for="cpu")` | `HardwareClassification.CPU` |
| `instantiate_device_type_tests(except_for=...)` | `HardwareClassification.ACCELERATOR` |
| `instantiate_device_type_tests(only_for="cuda")` | `HardwareClassification.CUDA` |
| `instantiate_device_type_tests(only_for="mps")` | `HardwareClassification.MPS` |
| `instantiate_device_type_tests(only_for="xpu")` | `HardwareClassification.XPU` |

**Structural contract** (mirrors the deterministic test linter):
- `GENERIC`: class NOT instantiated via `instantiate_device_type_tests`; methods take no `device`/`devices`.
- `ACCELERATOR`: class instantiated via `instantiate_device_type_tests`; every method takes `device`/`devices`; no `@only*` except `@onlyAccelerator`; the instantiate call uses no `only_for`.
- `CPU`/`CUDA`/`MPS`/`XPU`: class instantiated via `instantiate_device_type_tests` with `only_for=<device>`; every method takes `device`/`devices`; the instantiate call uses no `except_for`.
