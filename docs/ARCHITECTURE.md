# Architecture

This document describes the technical architecture of FLUX. It is the authoritative reference for module boundaries, data flow, asynchronous behavior, and extension points. For the reasoning behind these decisions, see [DESIGN_PHILOSOPHY.md](DESIGN_PHILOSOPHY.md).

---

## Table of Contents

1. [System Overview](#system-overview)
2. [The Three Phases](#the-three-phases)
3. [Asynchronous Data Flow](#asynchronous-data-flow)
4. [Execution Modes](#execution-modes)
5. [Fault Tolerance Model](#fault-tolerance-model)
6. [Extension Points](#extension-points)
7. [Comparison with RIPPLe](#comparison-with-ripple)
8. [Module Layout](#module-layout)

---

## System Overview

FLUX is structured as **three independent processing phases** connected by **bounded asynchronous queues**. Each phase is a self-contained module with a clearly defined input and output contract. No phase has direct knowledge of the others — they communicate exclusively through the queue layer.

```
                          ┌──────────────────────────────────────┐
                          │           Configuration              │
                          │       (YAML + Pydantic schema)       │
                          └────────────────┬─────────────────────┘
                                           │
                                           ▼
┌───────────────────┐  Queue A   ┌───────────────────┐  Queue B   ┌───────────────────┐
│                   │  ────────> │                   │  ────────> │                   │
│   Phase A         │   raw      │   Phase B         │  cleaned   │   Phase C         │
│   Extractor       │  frames    │   Preprocessor    │  tensors   │   ML Bridge       │
│                   │            │                   │            │                   │
│  • Butler / Mock  │            │  • 5 filters      │            │  • IterableDataset│
│  • PyVO catalogs  │            │  • normalization  │            │  • DataLoader     │
│  • async fetch    │            │  • channel ops    │            │  • prefetching    │
│  • retries        │            │  • cutout extract │            │  • batch assembly │
└───────────────────┘            └───────────────────┘            └───────────────────┘
         │                                │                                 │
         └────────────────────────────────┴─────────────────────────────────┘
                                          │
                                          ▼
                            ┌─────────────────────────────┐
                            │   Error Bus (structured)    │
                            │   • per-image failures      │
                            │   • retry policy            │
                            │   • metrics & logs          │
                            └─────────────────────────────┘
```

The system has three external surfaces: the **YAML configuration** (input contract), the **PyTorch DataLoader** produced by Phase C (output contract), and the **structured error bus** (observability contract). Everything else is internal and replaceable.

---

## The Three Phases

### Phase A — Extractor

**Responsibility:** Retrieve raw astronomical data from a configured source and emit individual frames into Queue A.

**Inputs:**
- A `SourceConfig` Pydantic model (validated YAML section)
- An optional query specification (sky region, filters, time range)

**Outputs:**
- `RawFrame` objects pushed into Queue A. Each `RawFrame` contains the image array, associated metadata (object ID, filter band, exposure parameters), and a provenance record (where it came from, when it was fetched).

**Internal structure:**
The Extractor is built around a `DataSource` abstract interface. Two concrete implementations ship with FLUX v0.1:

- `MockButlerSource` — implements the LSST Butler API contract over local files, allowing development and testing without telescope access. Reproduces the query semantics of `lsst.daf.butler` for the subset relevant to image retrieval.
- `GenericDirectorySource` — adapter for any flat directory of `.npy`, `.fits`, or HuggingFace-distributed datasets.

A future v0.2+ will add `LSSTButlerSource` (real Butler client) and `PyVOSource` (Virtual Observatory catalogs).

**Concurrency model:**
The Extractor uses `asyncio` for I/O concurrency. Multiple frames are fetched in parallel up to a configurable limit (`max_concurrent_fetches`), preventing memory blowup while keeping I/O saturated. Failed fetches are retried with exponential backoff before being dispatched to the error bus.

### Phase B — Preprocessor

**Responsibility:** Transform raw frames into clean, normalized tensors ready for ML consumption.

**Inputs:**
- `RawFrame` objects from Queue A
- A `PreprocessingConfig` Pydantic model specifying which filters and normalization to apply

**Outputs:**
- `ProcessedFrame` objects pushed into Queue B. Each contains a `torch.Tensor` (shape configurable, default `(C, H, W)`), the original metadata, and a record of which transformations were applied.

**Internal pipeline:**
The Preprocessor applies a configurable sequence of operations. Each operation implements the `Transform` interface and can be enabled, disabled, or reordered via YAML. The standard pipeline is:

```
RawFrame
   │
   ▼
[1] Cutout extraction        # (optional) extract region around object
   │
   ▼
[2] Noise filtering          # one or more of:
   │                         #   - Gaussian
   │                         #   - Median
   │                         #   - Sigma clipping
   │                         #   - Wavelet denoising
   │                         #   - Richardson-Lucy PSF deconvolution
   ▼
[3] Channel handling         # stacking, splitting, or filter selection
   │
   ▼
[4] Normalization            # Z-score, min-max, or per-channel
   │
   ▼
[5] Tensor conversion        # numpy → torch.Tensor
   │
   ▼
ProcessedFrame
```

The mathematical formulation of each filter is documented in [math/filters.md](math/filters.md). Normalization strategies are in [math/normalization.md](math/normalization.md). The Richardson-Lucy algorithm is detailed in [math/psf_deconvolution.md](math/psf_deconvolution.md).

**Concurrency model:**
In `simple` mode, transformations run sequentially per frame. In `fast` mode, the Preprocessor uses Dask delayed computation to parallelize across CPU cores, with chunking that respects memory constraints.

### Phase C — ML Bridge

**Responsibility:** Assemble processed frames into PyTorch-compatible batches and expose them as a standard `DataLoader`.

**Inputs:**
- `ProcessedFrame` objects from Queue B
- An `MLBridgeConfig` specifying batch size, prefetch depth, and target task (classification or super-resolution)

**Outputs:**
- A PyTorch `IterableDataset` instance, usable directly in any DeepLense training or inference loop without modification

**Internal structure:**
The ML Bridge implements `IterableDataset` rather than `Dataset` because the upstream queue is a stream, not a fixed-size collection. This is a deliberate architectural decision — it allows the pipeline to handle arbitrarily large surveys without ever materializing the full dataset in memory.

Two task adapters ship in v0.1:
- `ClassificationAdapter` — emits `(image_tensor, label)` pairs compatible with DeepLense Common Test models
- `SuperResolutionAdapter` — emits `(low_res, high_res)` pairs compatible with DeepLense SR models

**Prefetching:**
The ML Bridge maintains a configurable prefetch buffer (`prefetch_factor`, default 4) that pre-pulls batches from Queue B. This guarantees the GPU never waits — the next batch is always ready before the current one finishes its forward pass.

---

## Asynchronous Data Flow

The two queues between phases are bounded, backpressure-aware `asyncio.Queue` instances. Their bounded nature is critical: it prevents fast producers from overwhelming slow consumers.

### Queue Sizing

Both queues are sized according to the configuration:

```yaml
queues:
  queue_a_size: 64    # raw frames buffered between Extractor and Preprocessor
  queue_b_size: 32    # processed frames buffered between Preprocessor and ML Bridge
```

When a queue is full, the upstream producer awaits — this is normal backpressure, not an error. When a queue is empty, the downstream consumer awaits. The bounded size ensures memory usage is predictable regardless of dataset size.

### Lifecycle

```
1. Pipeline.run() is called
2. Configuration is validated by Pydantic
3. Three asyncio tasks are spawned: extractor_task, preprocessor_task, bridge_task
4. Each task runs its own loop, reading from its input queue and writing to its output
5. The Extractor signals end-of-stream via a sentinel value when its source is exhausted
6. The sentinel propagates: Preprocessor finishes flushing, then propagates to ML Bridge
7. The DataLoader's iterator raises StopIteration once all batches are consumed
```

### Cancellation

A pipeline can be cancelled at any point. Cancellation is propagated cooperatively: all three tasks receive a cancel signal, finish their current frame, drain their output queue, and exit cleanly. No orphan tasks. No leaked file handles.

---

## Execution Modes

FLUX ships with two execution modes selectable via a single YAML flag:

```yaml
execution:
  mode: simple   # or "fast"
```

### `simple` mode

- All three phases run sequentially in a single process
- No Dask, no asyncio internals exposed to the user
- Single command launches the pipeline
- Identical behavior to RIPPLe — chosen for users without distributed-computing experience
- Lower throughput, but trivially debuggable

### `fast` mode

- All three phases run as concurrent `asyncio` tasks
- Phase B uses Dask delayed for CPU parallelism within preprocessing
- Phase C uses prefetching to overlap data preparation with GPU computation
- Up to 4–8x throughput improvement on multi-core machines (preliminary estimate, to be benchmarked formally in [PERFORMANCE.md](PERFORMANCE.md))

The mode flag is the **only** difference visible to the user. The same YAML configuration runs identically in both modes, with the same outputs. This is the **maintainability contract**: a scientist can develop locally in `simple` mode and deploy to a cluster in `fast` mode without changing anything else.

---

## Fault Tolerance Model

FLUX is designed to never fail catastrophically on a single bad input. Failures are **localized** to the frame that caused them.

### Error categories

- **Recoverable I/O errors** — network timeouts, transient Butler unavailability. Handled via exponential backoff retries (configurable `max_retries`, default 3).
- **Data integrity errors** — corrupted files, invalid shapes, NaN values. Frame is dropped, error logged with full provenance, pipeline continues.
- **Configuration errors** — invalid YAML, missing fields, type mismatches. Pipeline fails fast at startup, before any data is fetched. This is intentional — a misconfigured pipeline should not silently produce garbage.
- **Out-of-memory errors** — non-recoverable. Logged and re-raised. Mitigated by bounded queues and Dask chunking, but ultimately a hardware concern.

### Error bus

All non-fatal errors flow into a structured **error bus** — a logger with JSON output that records:

```json
{
  "phase": "preprocessor",
  "frame_id": "DP0_g_band_00042",
  "error_type": "InvalidShapeError",
  "message": "Expected (3, 64, 64), got (3, 32, 32)",
  "retry_count": 0,
  "dropped": true,
  "timestamp": "2025-04-12T18:33:21Z"
}
```

This format is consumable by any standard log aggregation tool (Loki, ELK, simple `grep`). The error bus is the primary observability surface of FLUX.

---

## Extension Points

FLUX is designed to be extended without forking. The four formal extension points are:

### 1. New data source

Implement the `DataSource` interface:

```python
class DataSource(ABC):
    @abstractmethod
    async def fetch(self, query: SourceQuery) -> AsyncIterator[RawFrame]: ...
```

Register via the `sources` plugin registry in `flux/extractor/registry.py`. Reference implementations: `MockButlerSource`, `GenericDirectorySource`.

### 2. New preprocessing filter

Implement the `Transform` interface:

```python
class Transform(ABC):
    @abstractmethod
    def apply(self, frame: np.ndarray, metadata: dict) -> np.ndarray: ...
```

Register via the `transforms` plugin registry. Reference implementations: `GaussianFilter`, `MedianFilter`, `SigmaClipFilter`, `WaveletDenoiser`, `RichardsonLucyDeconvolver`.

### 3. New ML task adapter

Implement the `TaskAdapter` interface:

```python
class TaskAdapter(ABC):
    @abstractmethod
    def assemble_batch(self, frames: list[ProcessedFrame]) -> tuple[Tensor, ...]: ...
```

Reference implementations: `ClassificationAdapter`, `SuperResolutionAdapter`.

### 4. New configuration schema

Subclass the relevant Pydantic config model (`SourceConfig`, `PreprocessingConfig`, `MLBridgeConfig`) and register the subclass via `flux/config/registry.py`.

All extension points are documented with worked examples in [CONTRIBUTING.md](../CONTRIBUTING.md).

---

## Comparison with RIPPLe

RIPPLe (GSoC 2025) is the direct predecessor of FLUX. It established the foundational connection between LSST Butler and DeepLense and introduced YAML-based configuration. FLUX preserves the configuration philosophy and extends the architecture.

| Aspect | RIPPLe | FLUX |
|---|---|---|
| Architecture | Monolithic, sequential | Three decoupled async phases |
| I/O model | Blocking | Asynchronous with backpressure |
| Failure handling | Pipeline halts on error | Errors localized, pipeline continues |
| GPU utilization | Idle during fetch and preprocess | Continuous (prefetching) |
| Extensibility | Modify source code | Plugin registries |
| Execution modes | One | Two (`simple`, `fast`) |
| Configuration | YAML | YAML + Pydantic validation |
| Preprocessing filters | Limited | Five astronomy-grade filters |
| Super-resolution | Not directly supported | First-class via task adapters |
| Real LSST data | Synchronous fetch | Async-ready (mock now, real Butler in v0.2) |

FLUX is **not a rewrite** of RIPPLe. The YAML configuration layer is intentionally compatible, and a migration guide will be published with v0.2.

---

## Module Layout

The source code mirrors the architectural structure:

```
flux/
├── config/                # Pydantic models, YAML loading, validation
│   ├── schema.py
│   └── registry.py
├── extractor/             # Phase A
│   ├── base.py            # DataSource ABC
│   ├── mock_butler.py     # MockButlerSource
│   ├── generic.py         # GenericDirectorySource
│   └── registry.py
├── preprocessor/          # Phase B
│   ├── base.py            # Transform ABC
│   ├── filters/           # 5 filter implementations
│   ├── normalize.py
│   ├── cutout.py
│   └── registry.py
├── ml_bridge/             # Phase C
│   ├── base.py            # TaskAdapter ABC
│   ├── classification.py
│   ├── superres.py
│   └── dataset.py         # IterableDataset implementation
├── pipeline.py            # Orchestrates the three phases
├── queues.py              # Bounded async queue wrappers
├── errors.py              # Error bus + structured logging
└── __init__.py
```

Each subpackage has its own `__init__.py` exposing the public API. Internal helpers are prefixed with underscore. This layout is enforced by the import linter rules described in [CONTRIBUTING.md](../CONTRIBUTING.md).

---

## See also

- [DESIGN_PHILOSOPHY.md](DESIGN_PHILOSOPHY.md) — the principles behind these decisions
- [modules/extractor.md](modules/extractor.md) — full Phase A specification
- [modules/preprocessor.md](modules/preprocessor.md) — full Phase B specification
- [modules/ml_bridge.md](modules/ml_bridge.md) — full Phase C specification
- [PERFORMANCE.md](PERFORMANCE.md) — benchmark results and tuning guide