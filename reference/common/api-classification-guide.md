# API Classification Guide

How to classify device API calls during review. Shared by all modules.
This file owns the A/B/C category semantics; the lookup procedure below is
applied to the data in `reference/common/device-api-categories.yaml`.

**Primary references:**
- `reference/common/device-api-categories.yaml` — machine-readable data, the single source of truth
- `reference/common/backend-differences.md` — per-backend feature facts
- `reference/common/test-classification-standards.md` — the S1/S2/S3 strategy framework this feeds

## Classification Hierarchy

```
Category C (device-specific) > Category B (no wrapper) > Category A (replaceable) > none (CPU-only)
```

First match wins. A test using ANY Category C API is Strategy 3.

**S3 classification note:** An `if device_type == "<backend>"` conditional in
the test body does NOT make a test S3. Classify based on Category C API calls
only — check the catalog's Category C list.

## Category Semantics

| Category | Meaning | Strategy | Call-Site Action |
|----------|---------|----------|------------------|
| **A** | Has a `torch.accelerator.*` equivalent | S2 | Replace with `torch.accelerator.*` (drop-in if `same_name`, renamed if `name_differs`) |
| **B** | General cross-backend concept, no accelerator wrapper | S2 | No `torch.accelerator.*` replacement exists. Use the unified type where available (`torch.Stream`, `torch.Event`); otherwise keep the device-module call, guarded per `device_type` if needed |
| **C** | Truly device-specific | S3 | Keep as-is; the test stays on that device |

**Category B reconciliation:** the header comment in
`device-api-categories.yaml` ("category_b → Strategy 3: keep device-specific")
predates the current standards. The authoritative rule — per
`test-classification-standards.md` and the review checklist — is that B is S2
in *strategy* (the concept is cross-backend) while the *call site* stays on
the device module (no wrapper exists to replace it with). When the two
disagree, follow this guide, not the YAML header comment.

## Unified Stream and Event Types

Stream/event APIs (Category B) have unified replacements on the base
`torch` namespace:

| Device-module API | Unified replacement |
|-------------------|---------------------|
| `torch.cuda.stream` / `torch.xpu.stream` | `torch.Stream` |
| `torch.cuda.Stream` / `torch.xpu.Stream` | `torch.Stream` |
| `torch.cuda.Event` / `torch.xpu.Event` | `torch.Event` |
| `with torch.cuda.stream(s):` | `with torch.Stream(s):` |

Exception: inside dynamo-traced test cases, keep the device-module stream
APIs as-is — dynamo treats them as distinct operations from the unified
types, so both forms need separate testing. See
`reference/modules/graph/pitfalls.md`.

## Lookup Rules

### To classify an API call:

1. Check `category_a.same_name` — if the API name matches, it's Category A (replace with `torch.accelerator.<api>`)
2. Check `category_a.name_differs` — if the API name matches a `device_apis` entry, it's Category A (use the `accelerator_api` name)
3. Check `category_c.<backend>.*.apis` — if the full path matches, it's Category C (must stay device-specific)
4. Check `category_b.*` — if the API name matches, it's Category B (general concept: S2 strategy, keep the device-module call or use the unified type)
5. Otherwise — no device dependency (Strategy 1)

### Decision rules

API-call actions only; decorator-level rules live in
`test-classification-standards.md` (Blacklist vs. Whitelist Decorators).

- **Cat A same name**: Drop-in replace `torch.{device}` → `torch.accelerator`
- **Cat A name differs**: Use correct accelerator name (see `name_differs` section)
- **Cat B**: No accelerator replacement exists — use the unified type where available, otherwise keep the device-module call (the test itself is S2)
- **Cat C**: DO NOT REPLACE, keep original device module call

## YAML Quick Paths

```
category_a.same_name[].accelerator_api     → accelerator equivalents (same name)
category_a.name_differs[].accelerator_api  → accelerator equivalents (different name)
category_a.name_differs[].device_apis      → original device-specific names
category_a.accelerator_only[]              → accelerator-only APIs (no device equivalent)
category_b.<group>[].api                   → Category B APIs by functional group
category_c.<backend>.<group>.apis          → Category C APIs by backend and group
decision_rules                             → refactoring rules
architecture                               → dispatch chain, backend integration
```
