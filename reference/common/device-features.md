# Device-Specific Features

Per-backend feature differences that affect classification and skip decorators.
Human-readable summary of `reference/common/device-api-catalog.yaml`, which is
the machine-readable source of truth.

## Quick Reference

| Category | Rule | Strategy |
|----------|------|----------|
| **A** — Has `torch.accelerator` equivalent | Replace with `torch.accelerator.*` | Strategy 2 |
| **B** — General concept, no wrapper | Keep device-module call (unified type where available) | Strategy 2 |
| **C** — Truly device-specific | Must stay, never remove guards | Strategy 3 |

**Classification is hierarchical: C > B > A > none.** A test using ANY Cat C
API is Strategy 3.

## Decorator Rules

| Decorator | Action |
|-----------|--------|
| `@skipXPU`, `@skipCUDAIf`, `@skipMPS`, `@skipMeta` | **KEEP** (blacklist) |
| `@onlyNativeDeviceTypes`, `@onlyNativeDeviceTypesAnd` | **REMOVE** (redundant on device-agnostic classes) |
| `@onlyCUDA`, `@onlyOn` | **Enlarge** to `@onlyAccelerator` if test uses only Cat A/B APIs |

## Name Differences (most common mistakes)

| Device API | Accelerator API |
|------------|-----------------|
| `torch.cuda.current_device()` | `torch.accelerator.current_device_index()` |
| `torch.cuda.set_device()` | `torch.accelerator.set_device_index()` |
| `torch.cuda.device` (ctx mgr) | `torch.accelerator.device_index` (ctx mgr) |
| `torch.cuda.mem_get_info()` | `torch.accelerator.get_memory_info()` |
| `torch.cuda.CUDAGraph` | `torch.accelerator.Graph` |

## Backend Coverage

| Backend | Accelerator Coverage | Notes |
|---------|---------------------|-------|
| CUDA | High | Own Stream/Event/Graph classes |
| XPU | High | Own Stream/Event/Graph classes |
| MPS | Minimal (4 APIs) | No streams, no device switching |
| MTIA | Medium | Uses `torch.Stream`/`torch.Event` directly |

## Statistics

- **Cat A:** 21 API groups (8%) — replaceable
- **Cat B:** 46 API groups (19%) — candidates for abstraction
- **Cat C:** 182 API endpoints (73%) — truly device-specific
  - CUDA: 135 | XPU: 15 | MPS: 19 | MTIA: 13

## Sources

See `reference/common/device-api-catalog.yaml` for the full structured catalog
and source file references.
