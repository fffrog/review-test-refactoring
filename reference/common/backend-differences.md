# Backend Differences

Per-backend feature facts that affect classification and skip decorators.
Shared by all modules.

**Boundary:**
- Category semantics (what A/B/C mean): `reference/common/api-classification-guide.md`
- Decorator rules (keep/enlarge/remove): `reference/common/test-classification-standards.md`
- Data behind these facts: `reference/common/device-api-categories.yaml`

## Dtype Support

| Backend | Limitation |
|---------|------------|
| MPS | `float64` unsupported |
| MPS | `complex128` unsupported (double-precision complex) |
| MPS | Limited `torch.long`/int64 support in convolution and indexing ops |

Failure signature on MPS: `"Cannot convert a MPS Tensor to float64 dtype"` or
similar dtype conversion errors. The review procedure that consumes these
facts: `reference/common/review-checklist.md` (1b. Over-Generalization
Detection).

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

See `reference/common/device-api-categories.yaml` for the full structured
catalog and source file references.
