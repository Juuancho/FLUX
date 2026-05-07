# Module: ML Bridge (Phase C)

The ML Bridge is the third and final phase of the FLUX pipeline. It assembles processed frames into PyTorch-compatible batches and exposes them as a standard `IterableDataset`, ready to be consumed by any DeepLense training or inference loop.

For the role of the ML Bridge in the broader pipeline, see [ARCHITECTURE.md](../ARCHITECTURE.md). For the upstream phase that feeds it, see [modules/preprocessor.md](preprocessor.md).

---

## Table of Contents

1. [Responsibility](#responsibility)
2. [Public Interface](#public-interface)
3. [Data Types](#data-types)
4. [Built-in Task Adapters](#built-in-task-adapters)
5. [Prefetching and GPU Utilization](#prefetching-and-gpu-utilization)
6. [DataLoader Integration](#dataloader-integration)
7. [Configuration Reference](#configuration-reference)
8. [Adding a New Task Adapter](#adding-a-new-task-adapter)
9. [Testing Strategy](#testing-strategy)

---

## Responsibility

The ML Bridge has exactly one job: **assemble `ProcessedFrame` objects from Queue B into PyTorch batches and expose them as an `IterableDataset` consumable by a standard `DataLoader`.**

It is explicitly **not** responsible for:

- Training models — that belongs to DeepLense
- Computing losses, optimizers, or evaluation metrics — that belongs to DeepLense
- Augmenting data — augmentation belongs in Phase B as a `Transform` (per ordering rules in [preprocessor.md](preprocessor.md))
- Caching batches across epochs — the pipeline is a stream, not a cache

The output of this phase is a drop-in replacement for any PyTorch `DataLoader`. Existing DeepLense training scripts should require zero modification to consume it.

---

## Public Interface

The ML Bridge exposes one abstract base class for task-specific batch assembly and one orchestrator class.

```python
from abc import ABC, abstractmethod
from typing import Any
import torch
from flux.preprocessor.types import ProcessedFrame

class TaskAdapter(ABC):
    """Abstract interface for task-specific batch assembly."""

    @abstractmethod
    def assemble_batch(
        self,
        frames: list[ProcessedFrame],
    ) -> tuple[torch.Tensor, ...]:
        """Combine a list of ProcessedFrames into a model-ready batch.

        Implementations MUST:
          - Return a tuple of tensors matching the target model's signature.
          - Stack tensors along dim=0 (batch dimension).
          - Move tensors to the configured device only if device != 'cpu'.
          - Raise BatchAssemblyError on shape or dtype mismatches.
          - Be deterministic: same input list → same output tuple.
        """
        ...

    @property
    def output_signature(self) -> tuple[str, ...]:
        """Names of the tensors in the output tuple, e.g. ('image', 'label').
        Used for logging and debugging."""
        ...
```

Two design notes:

The return type is `tuple[torch.Tensor, ...]` rather than a dict because PyTorch `DataLoader` and most training loops expect positional unpacking (`for x, y in loader`). The `output_signature` property carries the names for logging without forcing dict semantics into the hot path.

Device placement is the responsibility of the adapter, not the orchestrator. Adapters that need GPU access must opt in explicitly via configuration. The default is CPU because moving to GPU prematurely wastes pinned memory.

---

## Data Types

### `BatchSpec`

Describes the shape and dtype contract a task adapter promises to produce. Used by `IterableDataset` to validate that frames can be batched together.

```python
from pydantic import BaseModel

class BatchSpec(BaseModel):
    tensor_shapes: list[tuple[int, ...]]    # one shape per output tensor
    tensor_dtypes: list[str]                # e.g. ["float32", "int64"]
    batch_dim: int = 0                      # always 0 in v0.1
```

Adapters declare their `BatchSpec` at construction time. Frames whose `ProcessedFrame.tensor.shape` does not match the declared shape are rejected before reaching `assemble_batch`, with a clear error message.

### `FluxIterableDataset`

The concrete `IterableDataset` produced by the ML Bridge. Subclasses `torch.utils.data.IterableDataset`:

```python
from torch.utils.data import IterableDataset

class FluxIterableDataset(IterableDataset):
    def __init__(
        self,
        queue: AsyncQueue,
        adapter: TaskAdapter,
        batch_size: int,
        prefetch_factor: int = 4,
    ):
        ...

    def __iter__(self):
        # Yields tuples of tensors as produced by the adapter
        ...
```

The `__iter__` method runs an internal asyncio bridge that pulls from the upstream queue, accumulates `batch_size` frames, calls `adapter.assemble_batch`, and yields the result. The bridge handles the impedance mismatch between async producers and sync consumers — PyTorch `DataLoader` is fundamentally synchronous, so the boundary must be crossed somewhere, and this is where.

---

## Built-in Task Adapters

FLUX v0.1 ships with two reference `TaskAdapter` implementations, covering the two model families currently maintained in DeepLense.

### `ClassificationAdapter`

Emits `(image, label)` pairs compatible with DeepLense Common Test classification models (Test I, Test V, and similar tasks).

```yaml
ml_bridge:
  task: classification
  batch_size: 32
  prefetch_factor: 4
  device: cpu
  classification:
    label_dtype: int64
    num_classes: 3
```

Output signature:

```python
output_signature = ("image", "label")
# image: shape (batch_size, C, H, W), dtype float32
# label: shape (batch_size,),         dtype int64
```

Frames missing a label (i.e., `ProcessedFrame.label is None`) are rejected with `MissingLabelError` before batch assembly. For inference workflows where labels are not available, use `InferenceAdapter` instead (planned for v0.2).

### `SuperResolutionAdapter`

Emits `(low_res, high_res)` pairs compatible with DeepLense super-resolution models.

```yaml
ml_bridge:
  task: super_resolution
  batch_size: 16
  prefetch_factor: 4
  device: cpu
  super_resolution:
    downsample_factor: 2
    downsample_method: bicubic
```

Output signature:

```python
output_signature = ("low_res", "high_res")
# low_res:  shape (batch_size, C, H/factor, W/factor), dtype float32
# high_res: shape (batch_size, C, H, W),               dtype float32
```

The `high_res` tensor is the original `ProcessedFrame.tensor`. The `low_res` tensor is derived on-the-fly via the configured downsampling method. Supported methods in v0.1: `bicubic`, `bilinear`, `nearest`. Custom downsamplers can be added by subclassing `SuperResolutionAdapter` and overriding `_downsample`.

---

## Prefetching and GPU Utilization

The ML Bridge maintains an internal **prefetch buffer** that pulls batches from Queue B before they are requested by the consumer. The buffer size is controlled by `prefetch_factor` (default 4): the bridge always tries to keep `prefetch_factor` batches ready ahead of the current iteration.

```yaml
ml_bridge:
  prefetch_factor: 4
```

The mechanism is straightforward: when `__iter__` starts, the bridge spawns an asyncio task that aggressively pulls and assembles batches into an internal `asyncio.Queue` of size `prefetch_factor`. The synchronous iterator then pops from that queue. While the GPU consumes batch N, batches N+1 through N+`prefetch_factor` are already in flight or completed.

Trade-offs of `prefetch_factor`:

- **Too low (1-2)**: GPU may stall waiting for the next batch, especially when Phase B is slow (heavy filters, large images).
- **Too high (>8)**: Memory consumption grows linearly with `prefetch_factor × batch_size × frame_size`. On a GTX 1650 with 4 GB VRAM, `prefetch_factor: 4` and `batch_size: 32` is a safe ceiling.
- **Default of 4**: empirically balances throughput and memory on consumer GPUs. Tuning guidance in [PERFORMANCE.md](../PERFORMANCE.md).

Prefetching is independent of execution mode. Even in `simple` mode, prefetching is active by default — it adds no architectural complexity for the user.

---

## DataLoader Integration

The ML Bridge exposes a factory method that wraps `FluxIterableDataset` in a standard `torch.utils.data.DataLoader`:

```python
from flux import Pipeline

pipeline = Pipeline.from_config("config.yaml")
loader = pipeline.as_dataloader()

for image, label in loader:
    # Standard PyTorch training loop
    ...
```

Internally, `as_dataloader` constructs:

```python
from torch.utils.data import DataLoader

DataLoader(
    dataset=flux_iterable_dataset,
    batch_size=None,           # None because IterableDataset already batches
    num_workers=0,             # 0 because asyncio handles concurrency
    pin_memory=config.pin_memory,
)
```

Two notes on these defaults:

`batch_size=None` is required when consuming an `IterableDataset` that already produces batched tensors. PyTorch raises an error if both the dataset and the DataLoader try to batch.

`num_workers=0` is required because the asyncio event loop driving the pipeline cannot be safely forked into worker processes. Increasing `num_workers` would create independent pipelines that compete for the same data source — almost always a bug. Users wanting CPU parallelism should configure it inside Phase B (`dask_scheduler`, `num_workers` in the preprocessor section), not in the DataLoader.

---

## Configuration Reference

A complete ML Bridge configuration in YAML:

```yaml
ml_bridge:
  task: classification
  batch_size: 32
  prefetch_factor: 4
  device: cpu
  pin_memory: true
  drop_last: false

  classification:
    label_dtype: int64
    num_classes: 3
```

Every field is validated by Pydantic at load time. The full schema reference, auto-generated from the Pydantic models, lives in [CONFIGURATION.md](../CONFIGURATION.md).

The `pin_memory: true` flag is recommended when the consumer is a CUDA model — it allows asynchronous host-to-device transfers, further reducing GPU idle time. It has no effect on CPU-only workflows.

The `drop_last: false` flag controls whether the final partial batch (containing fewer than `batch_size` frames) is yielded or discarded. Default is `false` because dropping data is rarely the right choice in inference; for training, users may prefer `true` to keep batch statistics consistent.

---

## Adding a New Task Adapter

Adding support for a new ML task (segmentation, regression, multi-task) follows the standard FLUX extension procedure.

**Step 1.** Create a new module under `flux/ml_bridge/` (e.g., `flux/ml_bridge/segmentation.py`).

**Step 2.** Define a configuration schema:

```python
from flux.config.base import TaskAdapterConfig
from typing import Literal

class SegmentationConfig(TaskAdapterConfig):
    task: Literal["segmentation"] = "segmentation"
    mask_dtype: str = "int64"
    num_classes: int
```

**Step 3.** Implement the adapter:

```python
from flux.ml_bridge.base import TaskAdapter, BatchSpec
import torch

class SegmentationAdapter(TaskAdapter):
    def __init__(self, config: SegmentationConfig):
        self.config = config

    def assemble_batch(self, frames):
        images = torch.stack([f.tensor for f in frames])
        masks  = torch.stack([f.metadata.extra["mask"] for f in frames])
        return (images, masks)

    @property
    def output_signature(self):
        return ("image", "mask")
```

**Step 4.** Register the adapter:

```python
# In flux/ml_bridge/registry.py
from .segmentation import SegmentationAdapter, SegmentationConfig

TASK_REGISTRY.register(
    name="segmentation",
    adapter_class=SegmentationAdapter,
    config_class=SegmentationConfig,
)
```

**Step 5.** Write contract tests by inheriting from `BaseTaskAdapterContractTest` (see [Testing Strategy](#testing-strategy)).

---

## Testing Strategy

The ML Bridge is tested at three levels:

**Unit tests** verify each adapter's batch assembly logic against synthetic `ProcessedFrame` lists. For `ClassificationAdapter`, this includes validating output shapes, dtypes, and label-tensor correctness. For `SuperResolutionAdapter`, this includes verifying the downsampling factor and method.

**Contract tests** verify that every concrete `TaskAdapter` satisfies the abstract contract: deterministic output, correct shape/dtype declarations, proper error types on malformed input. Parametrized over every registered adapter.

**Integration tests** verify end-to-end behavior with the rest of the pipeline: a small fixture flows from a `MockButlerSource` through a configured Preprocessor, into the ML Bridge, and out of a real `DataLoader`. Assertions check that the final batches match the declared `BatchSpec` and that `prefetch_factor` keeps the queue populated.

The test layout:

```
tests/
└── ml_bridge/
    ├── test_adapter_contract.py             # contract tests
    ├── test_classification_adapter.py
    ├── test_superresolution_adapter.py
    ├── test_iterable_dataset.py             # async-to-sync bridge
    ├── test_dataloader_integration.py
    └── test_prefetch_buffer.py
```

Coverage target for this module: **>88%**.

---

## See also

- [ARCHITECTURE.md](../ARCHITECTURE.md) — system-level view of the three phases
- [DESIGN_PHILOSOPHY.md](../DESIGN_PHILOSOPHY.md) — principles guiding adapter design
- [modules/extractor.md](extractor.md) — start of the pipeline
- [modules/preprocessor.md](preprocessor.md) — what feeds into the ML Bridge
- [CONFIGURATION.md](../CONFIGURATION.md) — full YAML schema reference
- [PERFORMANCE.md](../PERFORMANCE.md) — tuning prefetch and batch parameters