# Module: Preprocessor (Phase B)

The Preprocessor is the second and most complex phase of the FLUX pipeline. Its job is to transform raw astronomical frames into clean, normalized tensors ready for ML consumption. This document is the authoritative specification of its public interface, the five built-in transforms, the ordering rules, and the operational behavior.

For the role of the Preprocessor in the broader pipeline, see [ARCHITECTURE.md](../ARCHITECTURE.md). For the mathematical formulation of the filters and normalization strategies, see [math/filters.md](../math/filters.md), [math/normalization.md](../math/normalization.md), and [math/psf_deconvolution.md](../math/psf_deconvolution.md).

---

## Table of Contents

1. [Responsibility](#responsibility)
2. [Public Interface](#public-interface)
3. [Data Types](#data-types)
4. [The Standard Pipeline](#the-standard-pipeline)
5. [Built-in Transforms](#built-in-transforms)
6. [Ordering Rules](#ordering-rules)
7. [Concurrency Model](#concurrency-model)
8. [Error Handling](#error-handling)
9. [Configuration Reference](#configuration-reference)
10. [Adding a New Transform](#adding-a-new-transform)
11. [Testing Strategy](#testing-strategy)

---

## Responsibility

The Preprocessor has exactly one job: **transform `RawFrame` objects from Queue A into `ProcessedFrame` objects in Queue B, applying the configured chain of transformations to each frame.**

It is explicitly **not** responsible for:

- Fetching data — that is Phase A
- Assembling batches — that is Phase C
- Deciding which transformations are scientifically appropriate — that is the user's responsibility, encoded in the YAML configuration
- Modifying the original `RawFrame` — `RawFrame` is immutable; transformations produce new arrays

This phase is where the 2023 DeepLense Model IV failure originated. Real observational data was passed to models without proper noise handling, and the models collapsed. The Preprocessor exists to ensure that does not happen again.

---

## Public Interface

The Preprocessor exposes one abstract base class for individual transformations and one orchestrator class that composes them.

```python
from abc import ABC, abstractmethod
import numpy as np
from flux.preprocessor.types import TransformContext

class Transform(ABC):
    """Abstract interface for any preprocessing transformation."""

    @abstractmethod
    def apply(
        self,
        image: np.ndarray,
        context: TransformContext,
    ) -> np.ndarray:
        """Apply the transformation to a single image array.

        Implementations MUST:
          - Be pure functions: same input → same output, no global state.
          - Not modify the input array (use copies if needed).
          - Preserve numerical dtype unless the transform explicitly changes it.
          - Raise TransformError on unrecoverable failures.
          - Be safe to call from multiple threads (Dask requires this).
        """
        ...

    @property
    def name(self) -> str:
        """Stable identifier used in logs and provenance records."""
        return self.__class__.__name__

    def estimate_memory_overhead(self, input_shape: tuple) -> int:
        """Return expected peak memory in bytes during apply(),
        or 0 if negligible. Used by Dask scheduler for chunking."""
        return 0
```

The `apply` method takes a `TransformContext` rather than just the image. The context carries the `FrameMetadata` and `Provenance` from the original `RawFrame`, plus a mutable `transform_log` dict that each transform appends to. This is how the final `ProcessedFrame` knows exactly which transformations were applied and with which parameters.

Two design notes:

The purity requirement (**same input → same output, no global state**) is enforced by the contract test suite. Transforms that depend on external state (random number generators, mutable shared variables) violate this contract and break Dask parallelization.

The `estimate_memory_overhead` method matters for `fast` mode. PSF deconvolution, for example, allocates working arrays roughly 4x the size of the input. Returning that estimate lets Dask choose chunk sizes that do not OOM the worker.

---

## Data Types

### `TransformContext`

```python
from pydantic import BaseModel, ConfigDict
from typing import Any

class TransformContext(BaseModel):
    model_config = ConfigDict(arbitrary_types_allowed=True)

    metadata: FrameMetadata        # carried forward from RawFrame
    provenance: Provenance         # carried forward from RawFrame
    transform_log: list[dict] = []  # mutable; each transform appends an entry
```

Each entry in `transform_log` records:

```python
{
  "name": "GaussianFilter",
  "applied_at": datetime.utcnow(),
  "parameters": {"sigma": 1.5, "kernel_size": 7},
  "input_shape": (3, 64, 64),
  "output_shape": (3, 64, 64),
  "duration_ms": 2.3,
}
```

### `ProcessedFrame`

The unit of data that crosses the boundary from Phase B to Phase C. Defined as a frozen Pydantic model:

```python
import torch

class ProcessedFrame(BaseModel):
    model_config = ConfigDict(arbitrary_types_allowed=True, frozen=True)

    frame_id: str                  # inherited from RawFrame
    tensor: torch.Tensor           # final preprocessed tensor
    label: int | None              # task-dependent, may be None
    metadata: FrameMetadata        # inherited
    provenance: Provenance         # inherited (with retry_count from extractor)
    transform_log: list[dict]      # full audit trail
```

The tensor's shape, dtype, and device are determined by the configuration. The default for classification is `(3, H, W)` `float32` on CPU; the ML Bridge moves it to GPU as needed.

---

## The Standard Pipeline

The Preprocessor applies a configurable sequence of transforms in a fixed conceptual order. Each step is **optional and configurable** but the order is **enforced** — see [Ordering Rules](#ordering-rules) for why.

```
RawFrame.image (numpy ndarray, raw)
   │
   ▼
[1] Cutout extraction         ─── extract region around target object
   │                              (skipped if input is already a cutout)
   ▼
[2] Noise filtering chain     ─── one or more of:
   │                              • Gaussian
   │                              • Median
   │                              • Sigma clipping
   │                              • Wavelet denoising
   │                              • Richardson-Lucy PSF deconvolution
   ▼
[3] Channel handling          ─── stack, split, select, or reorder filter bands
   │
   ▼
[4] Normalization             ─── Z-score, min-max, or per-channel
   │
   ▼
[5] Tensor conversion         ─── numpy → torch.Tensor with target dtype/device
   │
   ▼
ProcessedFrame.tensor
```

---

## Built-in Transforms

FLUX v0.1 ships with **five noise-handling filters** and **three normalization strategies**. The mathematical details are in the `math/` documentation; this section specifies the API contract.

### 1. `GaussianFilter`

```yaml
- type: GaussianFilter
  sigma: 1.0           # standard deviation in pixels
  kernel_size: 5       # must be odd; 0 = auto-derive from sigma
  mode: reflect        # boundary handling: reflect, constant, nearest
```

Used for general-purpose smoothing of small-scale noise. Cheap, well-understood, almost always safe. See [math/filters.md](../math/filters.md#gaussian-filter) for the convolution kernel formulation.

### 2. `MedianFilter`

```yaml
- type: MedianFilter
  kernel_size: 3       # must be odd
  mode: reflect
```

Specifically targeted at salt-and-pepper noise from dead pixels in CCD sensors — a common artifact in real telescope data. Preserves edges better than Gaussian for impulse noise. See [math/filters.md](../math/filters.md#median-filter).

### 3. `SigmaClipFilter`

```yaml
- type: SigmaClipFilter
  sigma_threshold: 3.0    # clip pixels beyond this many std deviations
  iterations: 3           # number of refinement passes
  replace_with: median    # what to substitute clipped pixels: median, zero, mean
```

The standard astronomical technique for outlier rejection. Iteratively computes pixel statistics, identifies values beyond `sigma_threshold` standard deviations, and replaces them. Effective against cosmic ray hits and saturated pixels. See [math/filters.md](../math/filters.md#sigma-clipping).

### 4. `WaveletDenoiser`

```yaml
- type: WaveletDenoiser
  wavelet: db4            # wavelet basis: db4, db8, sym4, coif1
  level: 3                # decomposition levels
  threshold_mode: soft    # soft or hard thresholding
  threshold_method: bayes # bayes, visu, or fixed
  fixed_threshold: 0.1    # only used if threshold_method = fixed
```

Wavelet denoising preserves fine structures (such as the thin arcs of strong gravitational lenses) better than Fourier-domain methods, which is why it is preferred in lens detection pipelines. See [math/filters.md](../math/filters.md#wavelet-denoising) for the multi-resolution decomposition theory.

### 5. `RichardsonLucyDeconvolver`

```yaml
- type: RichardsonLucyDeconvolver
  psf_path: ./calibration/psf_g_band.fits
  iterations: 30
  regularization: tv      # tv (total variation), none, or tikhonov
  regularization_weight: 0.01
```

Corrects the blurring introduced by the atmosphere and the telescope's optical system. This is the most expensive filter in the pipeline (iterative, ~30 passes per image) and the most powerful — it can recover detail lost to seeing. The full algorithm derivation is in [math/psf_deconvolution.md](../math/psf_deconvolution.md).

The PSF (Point Spread Function) must be provided as a calibration file matching the filter band of the input data. FLUX ships with reference PSFs for the standard LSST bands derived from the DP0 simulation specifications.

### Normalization Strategies

```yaml
normalization:
  type: ZScore          # ZScore, MinMax, or PerChannelZScore
  stats_source: dataset # dataset (Welford online), per_image, or fixed
  fixed_mean: null      # required if stats_source = fixed
  fixed_std: null       # required if stats_source = fixed
```

- **ZScore** — global mean and std, computed once over the dataset using Welford's online algorithm. Standard for ImageNet-style transfer learning.
- **MinMax** — scale to [0, 1] per image. Useful when absolute pixel values are not meaningful.
- **PerChannelZScore** — separate Z-score per filter band. Recommended for multi-band astronomical data where different bands have very different brightness distributions.

Mathematical formulation in [math/normalization.md](../math/normalization.md).

---

## Ordering Rules

The five filtering steps **must** run in a specific conceptual order, even when several filters are combined. The Preprocessor enforces this order regardless of the order they appear in the YAML configuration. This is intentional — silently respecting a wrong user-specified order is worse than overriding it.

The enforced order is:

```
SigmaClipFilter → MedianFilter → RichardsonLucyDeconvolver → GaussianFilter → WaveletDenoiser
```

The reasoning, in scientific terms:

1. **Sigma clipping first.** Outliers (cosmic rays, saturated pixels) must be removed before any operation that uses pixel statistics, because outliers contaminate mean/std calculations. Running sigma clip after any other filter means those filters have already smeared the outliers across neighbor pixels.

2. **Median next.** Median handles impulse noise (dead pixels) that sigma clipping does not catch (because dead pixels are zero, not outliers). It must run before deconvolution, because deconvolution amplifies any remaining impulse noise dramatically.

3. **Deconvolution third.** PSF deconvolution operates on a "clean" image — it assumes the noise is approximately Gaussian. Running it after sigma clip + median ensures that assumption holds. Running it first would amplify cosmic rays into ringing artifacts across the entire frame.

4. **Gaussian smoothing fourth.** After deconvolution, fine high-frequency noise is amplified. A light Gaussian smooth tames this without losing the structure deconvolution recovered.

5. **Wavelet denoising last.** Wavelet methods are sophisticated noise removers that work best on data that is already "well-behaved." They are the polish, not the foundation.

The user can disable any of the five filters individually. The user **cannot** reorder them. If a use case genuinely requires a different order, that is a signal to extend FLUX with a custom ordered pipeline via the [RFC process](../RFC_PROCESS.md), not to subvert this constraint.

After the noise filtering chain, **channel handling** runs, then **normalization**, then **tensor conversion**. These three are also fixed in order: you cannot normalize before deciding what channels you have, and you cannot convert to a tensor before normalization without losing dtype precision.

---

## Concurrency Model

The Preprocessor's concurrency depends on the execution mode:

**`simple` mode.** Transformations run sequentially, one frame at a time. The async loop pulls a `RawFrame` from Queue A, applies the configured chain, pushes the `ProcessedFrame` to Queue B, repeat. No threads, no Dask, no parallelism within preprocessing. This is slower but trivially debuggable.

**`fast` mode.** Multiple frames are processed in parallel using Dask delayed computation. The Preprocessor wraps each `apply` call in a Dask delayed task and submits batches to the Dask scheduler. The scheduler chooses the worker pool (threads or processes) based on configuration:

```yaml
preprocessor:
  dask_scheduler: threads    # threads, processes, or single-threaded
  num_workers: 0             # 0 = auto-detect (cpu_count - 1)
  chunk_size: 16             # frames per Dask task
```

Threads are the default because most filters are NumPy operations that release the GIL. Processes are available for filters that do not — but at a cost (frame data is serialized between workers).

The `chunk_size` parameter balances scheduling overhead (small chunks → too many tasks) against load balancing (large chunks → workers finish at different times). The default of 16 is a good starting point for typical workloads; tuning guidance is in [PERFORMANCE.md](../PERFORMANCE.md).

A critical invariant: **regardless of mode, the order of frames at the output of Phase B is not guaranteed to match the input order**. Frames are tagged by `frame_id` and downstream consumers must not rely on ordering. This is a deliberate consequence of parallelism — enforcing order would serialize the stage and erase the speedup.

---

## Error Handling

The Preprocessor classifies failures into three categories:

**Recoverable per-frame errors.** The frame is malformed (wrong shape, contains only NaN, etc.) but the pipeline can continue with other frames. The frame is dropped, an entry is emitted to the error bus, and the next frame is processed. Examples: `InvalidShapeError`, `AllNaNFrameError`, `EmptyChannelError`.

**Transform configuration errors.** A transform was instantiated with invalid parameters (e.g., `kernel_size: -3`, `sigma: 0`). These are caught at startup by Pydantic validation and never reach runtime. If they somehow do, the pipeline halts — a misconfigured transform is a programming error, not a data error.

**Catastrophic errors.** Out of memory, missing PSF calibration file, Dask worker died. Pipeline halts and the error propagates to the user. These are not the Preprocessor's responsibility to recover from.

Per-frame errors are emitted with full context:

```json
{
  "phase": "preprocessor",
  "frame_id": "DP0_g_band_00042",
  "transform_at_failure": "RichardsonLucyDeconvolver",
  "transforms_completed": ["SigmaClipFilter", "MedianFilter"],
  "error_type": "AllNaNFrameError",
  "message": "Frame contains only NaN after PSF deconvolution",
  "dropped": true,
  "timestamp": "2025-04-12T18:33:21Z"
}
```

---

## Configuration Reference

A complete preprocessor configuration in YAML:

```yaml
preprocessor:
  cutout:
    enabled: true
    size: [64, 64]
    center_strategy: peak    # peak, metadata, or coords

  filters:
    - type: SigmaClipFilter
      sigma_threshold: 3.0
      iterations: 3
      replace_with: median

    - type: MedianFilter
      kernel_size: 3

    - type: RichardsonLucyDeconvolver
      psf_path: ./calibration/psf_g_band.fits
      iterations: 30
      regularization: tv
      regularization_weight: 0.01

    - type: GaussianFilter
      sigma: 0.8
      kernel_size: 5

  channel_handling:
    strategy: stack          # stack, split, select, or reorder
    selected_bands: [g, r, i]

  normalization:
    type: PerChannelZScore
    stats_source: dataset

  tensor:
    dtype: float32
    layout: CHW              # CHW or HWC

  dask_scheduler: threads
  num_workers: 0
  chunk_size: 16
```

Every field is validated by Pydantic at load time. Unknown fields cause validation errors. The full schema reference, auto-generated from the Pydantic models, lives in [CONFIGURATION.md](../CONFIGURATION.md).

---

## Adding a New Transform

Adding a new noise filter or normalization strategy is a common extension. The procedure parallels the Extractor's:

**Step 1.** Create a new module under `flux/preprocessor/filters/` (e.g., `flux/preprocessor/filters/my_denoiser.py`).

**Step 2.** Define a configuration schema:

```python
from flux.config.base import TransformConfig
from typing import Literal

class MyDenoiserConfig(TransformConfig):
    type: Literal["MyDenoiser"] = "MyDenoiser"
    strength: float = 0.5
    preserve_edges: bool = True
```

**Step 3.** Implement the transform:

```python
from flux.preprocessor.base import Transform
import numpy as np

class MyDenoiser(Transform):
    def __init__(self, config: MyDenoiserConfig):
        self.config = config

    def apply(self, image, context):
        # Pure function: same input → same output, no side effects
        result = self._denoise(image, self.config.strength)
        return result

    def estimate_memory_overhead(self, input_shape):
        return int(np.prod(input_shape) * 4 * 2)  # 2x input size
```

**Step 4.** Register the transform:

```python
# In flux/preprocessor/registry.py
from .filters.my_denoiser import MyDenoiser, MyDenoiserConfig

TRANSFORM_REGISTRY.register(
    name="MyDenoiser",
    transform_class=MyDenoiser,
    config_class=MyDenoiserConfig,
    ordering_slot="wavelet",  # which slot in the ordering rules
)
```

The `ordering_slot` field tells FLUX where to insert your filter in the [enforced order](#ordering-rules). Valid slots are `sigma_clip`, `median`, `deconvolution`, `gaussian`, `wavelet`. Choosing the right slot is a scientific decision documented in your transform's docstring.

**Step 5.** Write contract tests:

The base test class `tests/preprocessor/test_transform_contract.py` runs every concrete transform against a standard suite (purity, dtype preservation, no input mutation, error propagation). Inherit from `BaseTransformContractTest` and parametrize with your config.

---

## Testing Strategy

The Preprocessor is the most heavily tested module in FLUX. Three test layers:

**Unit tests** verify each filter against synthetic inputs with known ground truth. For `GaussianFilter`, this means testing that a delta function input produces a discretized Gaussian output. For `SigmaClipFilter`, this means testing that a known outlier is correctly identified. Each filter has its own test file with mathematical correctness checks.

**Contract tests** verify that every concrete `Transform` satisfies the abstract contract: purity, no input mutation, dtype preservation, exception types. Parametrized over every registered transform.

**Integration tests** verify end-to-end behavior of the chain: a `RawFrame` enters Queue A, the configured chain runs, a `ProcessedFrame` exits Queue B with the expected `transform_log`. These tests use small fixture datasets and run in `simple` mode for determinism.

**Numerical regression tests** lock in the exact output of each filter on reference inputs. If a transform's output changes by more than a documented epsilon, the test fails. This catches subtle bugs that contract tests miss (e.g., off-by-one in convolution boundary handling).

The test layout:

```
tests/
└── preprocessor/
    ├── test_transform_contract.py           # contract tests
    ├── filters/
    │   ├── test_gaussian.py                 # math correctness
    │   ├── test_median.py
    │   ├── test_sigma_clip.py
    │   ├── test_wavelet.py
    │   └── test_richardson_lucy.py
    ├── test_normalization.py
    ├── test_ordering_rules.py               # the order is enforced
    ├── test_preprocessor_integration.py
    └── fixtures/
        ├── synthetic_lens.npy
        ├── synthetic_noise.npy
        └── reference_psf.fits
```

---

## See also

- [ARCHITECTURE.md](../ARCHITECTURE.md) — system-level view of the three phases
- [DESIGN_PHILOSOPHY.md](../DESIGN_PHILOSOPHY.md) — principles guiding transform design
- [modules/extractor.md](extractor.md) — what feeds into the Preprocessor
- [modules/ml_bridge.md](ml_bridge.md) — what consumes the Preprocessor output
- [math/filters.md](../math/filters.md) — mathematical formulation of all five filters
- [math/normalization.md](../math/normalization.md) — Welford's algorithm and normalization variants
- [math/psf_deconvolution.md](../math/psf_deconvolution.md) — Richardson-Lucy derivation
- [CONFIGURATION.md](../CONFIGURATION.md) — full YAML schema reference
- [PERFORMANCE.md](../PERFORMANCE.md) — tuning Dask parameters