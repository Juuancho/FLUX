# Configuration Reference

This document is the authoritative reference for the FLUX YAML configuration schema. Every field, every default, every validation rule is documented here.

The configuration system is built on Pydantic, which means two things: every field is type-checked at load time, and unknown fields cause validation errors rather than being silently ignored. A misconfigured pipeline never reaches runtime.

For the design rationale behind this approach, see [DESIGN_PHILOSOPHY.md](DESIGN_PHILOSOPHY.md#validation-at-the-boundary).

---

## Table of Contents

1. [File Format and Loading](#file-format-and-loading)
2. [Top-Level Structure](#top-level-structure)
3. [Pipeline Configuration](#pipeline-configuration)
4. [Source Configuration](#source-configuration)
5. [Preprocessor Configuration](#preprocessor-configuration)
6. [ML Bridge Configuration](#ml-bridge-configuration)
7. [Logging and Error Bus](#logging-and-error-bus)
8. [Environment Variable Substitution](#environment-variable-substitution)
9. [Validation Rules](#validation-rules)
10. [Complete Examples](#complete-examples)

---

## File Format and Loading

FLUX configurations are YAML files following the YAML 1.2 specification. Configuration files conventionally use the `.yaml` extension and live in a `configs/` directory at the root of the project.

A configuration is loaded into a `Pipeline` instance through the `from_config` factory method:

```python
from flux import Pipeline

pipeline = Pipeline.from_config("configs/classification_demo.yaml")
```

At load time, FLUX performs the following steps in order:

1. Read the YAML file from disk
2. Substitute environment variables in the form `${VAR_NAME}` (see [below](#environment-variable-substitution))
3. Validate the parsed structure against the Pydantic schema
4. Resolve plugin types (e.g. `MockButlerSource` → registered class)
5. Instantiate phase objects with their validated configurations

If any step fails, the pipeline is not created and the user receives a precise error message identifying the file, the line, and the field that caused the failure.

---

## Top-Level Structure

Every FLUX configuration has the following top-level structure:

```yaml
pipeline:           # Pipeline orchestration: execution mode, queues, retries
  ...

source:             # Phase A: Extractor configuration
  ...

preprocessor:       # Phase B: Preprocessor configuration
  ...

ml_bridge:          # Phase C: ML Bridge configuration
  ...

logging:            # Optional: structured logging and error bus
  ...
```

All four primary sections (`pipeline`, `source`, `preprocessor`, `ml_bridge`) are required. The `logging` section is optional and falls back to sensible defaults when omitted.

---

## Pipeline Configuration

Controls the orchestration of the three phases.

```yaml
pipeline:
  mode: simple                  # or "fast"
  queue_a_size: 64
  queue_b_size: 32
  shutdown_timeout_seconds: 30
```

| Field | Type | Default | Description |
|---|---|---|---|
| `mode` | `Literal["simple", "fast"]` | required | Execution mode. `simple` runs phases sequentially, `fast` runs them concurrently with Dask. See [ARCHITECTURE.md](ARCHITECTURE.md#execution-modes). |
| `queue_a_size` | `int >= 1` | `64` | Maximum frames buffered between Extractor and Preprocessor. Larger values increase memory; smaller values increase backpressure. |
| `queue_b_size` | `int >= 1` | `32` | Maximum processed frames buffered between Preprocessor and ML Bridge. Typically smaller than `queue_a_size` because processed tensors are larger than raw frames. |
| `shutdown_timeout_seconds` | `float > 0` | `30.0` | Maximum time to wait for clean shutdown before forcing termination. |

---

## Source Configuration

The `source` section configures Phase A. The `type` field selects a registered source implementation; the remaining fields are validated against that source's specific schema.

### Common fields

These fields are accepted by every source type:

```yaml
source:
  type: <SourceType>            # required
  limit: null                   # optional: maximum frames to fetch
  max_concurrent_fetches: 8
  max_retries: 3
  initial_backoff_seconds: 1.0
  backoff_multiplier: 2.0
```

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `string` | required | Name of a source registered in `flux/extractor/registry.py`. Built-in: `MockButlerSource`, `GenericDirectorySource`. |
| `limit` | `int > 0` or `null` | `null` | Maximum number of frames to fetch. `null` means unlimited (consume the entire query result). |
| `max_concurrent_fetches` | `int >= 1` | `8` | Maximum simultaneous fetches. Higher values saturate I/O but increase memory. |
| `max_retries` | `int >= 0` | `3` | Number of retries on transient errors before giving up on a frame. `0` disables retries. |
| `initial_backoff_seconds` | `float > 0` | `1.0` | Wait time before the first retry. |
| `backoff_multiplier` | `float >= 1` | `2.0` | Exponential backoff factor. Wait time before retry $k$ is `initial_backoff * multiplier^k`. |

### `MockButlerSource`

A standalone implementation that mimics the LSST Butler API over local files. Used for development, testing, and demonstrations without telescope access.

```yaml
source:
  type: MockButlerSource
  collection: DP0_mock
  filter_bands: [g, r, i]
  tract: 4225
  patch: "3,4"
  manifest_path: ./data/mock_butler/manifest.yaml
  limit: 1000
```

| Field | Type | Default | Description |
|---|---|---|---|
| `collection` | `string` | required | Butler collection name. In the mock, this maps to a top-level key in the manifest. |
| `filter_bands` | `list[string]` | required | Filter bands to retrieve. Standard LSST values: `g`, `r`, `i`, `z`, `y`. |
| `tract` | `int` | required | Sky tract identifier. |
| `patch` | `string` | required | Patch identifier within the tract, formatted as `"<i>,<j>"`. |
| `manifest_path` | `path` | required | Path to the manifest YAML mapping Butler queries to local files. |

### `GenericDirectorySource`

A simpler adapter for any flat directory of supported files. Useful for HuggingFace-distributed datasets and local data that does not need Butler-shaped queries.

```yaml
source:
  type: GenericDirectorySource
  root_path: ./data/deeplense_classification
  extensions: [".npy"]
  label_from_subfolder: true
  recursive: true
  limit: null
```

| Field | Type | Default | Description |
|---|---|---|---|
| `root_path` | `path` | required | Root directory containing the data files. Must exist at load time. |
| `extensions` | `list[string]` | `[".npy"]` | File extensions to include. Files with other extensions are ignored. Supported: `.npy`, `.fits`. |
| `label_from_subfolder` | `bool` | `false` | If `true`, the immediate parent directory name is used as the integer-mapped class label. Required for classification tasks. |
| `recursive` | `bool` | `true` | If `true`, walks subdirectories recursively. If `false`, only files directly in `root_path` are considered. |

---

## Preprocessor Configuration

The `preprocessor` section configures Phase B. It has four sub-sections: `cutout`, `filters`, `channel_handling`, and `normalization`, plus mode-specific scheduler parameters.

### Cutout

Optional first stage that extracts a fixed-size region around a target. Skipped when input is already a cutout.

```yaml
preprocessor:
  cutout:
    enabled: false
    size: [64, 64]
    center_strategy: peak       # peak, metadata, or coords
    coords:                     # required if center_strategy = coords
      x: 32
      y: 32
```

| Field | Type | Default | Description |
|---|---|---|---|
| `enabled` | `bool` | `false` | If `false`, skip cutout extraction entirely. |
| `size` | `[int, int]` | `[64, 64]` | Output cutout dimensions in pixels, `[height, width]`. |
| `center_strategy` | `Literal["peak", "metadata", "coords"]` | `peak` | How to determine the cutout center. `peak`: brightest pixel. `metadata`: from `FrameMetadata.ra/dec`. `coords`: explicit `coords` field. |
| `coords` | `{x: int, y: int}` | required if `center_strategy = coords` | Explicit center coordinates. |

### Filters

Ordered list of noise filters. Note: regardless of the order in this list, FLUX enforces the canonical filter order documented in [modules/preprocessor.md](modules/preprocessor.md#ordering-rules). Each filter type is documented in [math/filters.md](math/filters.md).

```yaml
preprocessor:
  filters:
    - type: SigmaClipFilter
      sigma_threshold: 3.0
      iterations: 3
      replace_with: median

    - type: MedianFilter
      kernel_size: 3
      mode: reflect

    - type: RichardsonLucyDeconvolver
      psf_path: ./calibration/psf_g_band.fits
      iterations: 30
      regularization: tv
      regularization_weight: 0.01

    - type: GaussianFilter
      sigma: 0.8
      kernel_size: 0              # 0 = auto-derive from sigma

    - type: WaveletDenoiser
      wavelet: db4
      level: 3
      threshold_mode: soft
      threshold_method: bayes
```

#### `SigmaClipFilter`

| Field | Type | Default | Description |
|---|---|---|---|
| `sigma_threshold` | `float > 0` | `3.0` | Pixels deviating by more than this many standard deviations are flagged. |
| `iterations` | `int >= 1` | `3` | Maximum refinement passes. The algorithm may converge earlier. |
| `replace_with` | `Literal["median", "mean", "zero"]` | `median` | Substitution strategy for flagged pixels. |

#### `MedianFilter`

| Field | Type | Default | Description |
|---|---|---|---|
| `kernel_size` | `odd int >= 3` | `3` | Window size in pixels. Must be odd. |
| `mode` | `Literal["reflect", "constant", "nearest"]` | `reflect` | Boundary handling. |

#### `RichardsonLucyDeconvolver`

| Field | Type | Default | Description |
|---|---|---|---|
| `psf_path` | `path` | required | Path to a PSF calibration file (`.fits` or `.npy`). The PSF must be centered, normalized, and match the filter band of the data. |
| `iterations` | `int >= 1` | `30` | Number of Richardson-Lucy iterations. See [math/psf_deconvolution.md](math/psf_deconvolution.md#convergence-and-stopping). |
| `regularization` | `Literal["none", "tv", "tikhonov"]` | `tv` | Regularization strategy. |
| `regularization_weight` | `float >= 0` | `0.01` | Strength of the regularization penalty. Ignored when `regularization = none`. |

#### `GaussianFilter`

| Field | Type | Default | Description |
|---|---|---|---|
| `sigma` | `float > 0` | `1.0` | Standard deviation of the Gaussian kernel in pixels. |
| `kernel_size` | `odd int >= 0` | `0` | Kernel size. `0` means auto-derive as `2 * ceil(3 * sigma) + 1`. |
| `mode` | `Literal["reflect", "constant", "nearest"]` | `reflect` | Boundary handling. |

#### `WaveletDenoiser`

| Field | Type | Default | Description |
|---|---|---|---|
| `wavelet` | `Literal["db4", "db8", "sym4", "coif1"]` | `db4` | Wavelet basis. |
| `level` | `int >= 1` | `3` | Decomposition levels. |
| `threshold_mode` | `Literal["soft", "hard"]` | `soft` | Thresholding function. |
| `threshold_method` | `Literal["bayes", "visu", "fixed"]` | `bayes` | Threshold selection method. See [math/filters.md](math/filters.md#threshold-selection). |
| `fixed_threshold` | `float > 0` | required if `threshold_method = fixed` | User-specified threshold value. |

### Channel handling

Controls how filter bands are combined into the output tensor.

```yaml
preprocessor:
  channel_handling:
    strategy: stack             # stack, split, select, or reorder
    selected_bands: [g, r, i]
```

| Field | Type | Default | Description |
|---|---|---|---|
| `strategy` | `Literal["stack", "split", "select", "reorder"]` | `stack` | How to combine bands. `stack`: concatenate as channels. `split`: emit one frame per band. `select`: keep only `selected_bands`. `reorder`: rearrange to match `selected_bands` order. |
| `selected_bands` | `list[string]` | required for `select`/`reorder` | Bands to keep, in the desired order. |

### Normalization

Applies a normalization strategy to the preprocessed tensor. See [math/normalization.md](math/normalization.md) for the mathematical formulation.

```yaml
preprocessor:
  normalization:
    type: PerChannelZScore
    stats_source: dataset
    fixed_mean: null
    fixed_std: null
```

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `Literal["ZScore", "MinMax", "PerChannelZScore"]` | `PerChannelZScore` | Normalization variant. |
| `stats_source` | `Literal["dataset", "per_image", "fixed"]` | `dataset` | Where statistics come from. `dataset` uses Welford's online algorithm. `per_image` recomputes per frame. `fixed` uses the provided values. |
| `fixed_mean` | `float` or `list[float]` | required if `stats_source = fixed` | Mean for normalization. List required for `PerChannelZScore`. |
| `fixed_std` | `float > 0` or `list[float > 0]` | required if `stats_source = fixed` | Standard deviation for normalization. |

### Tensor conversion

```yaml
preprocessor:
  tensor:
    dtype: float32              # float32 or float64
    layout: CHW                 # CHW or HWC
```

| Field | Type | Default | Description |
|---|---|---|---|
| `dtype` | `Literal["float32", "float64"]` | `float32` | Output tensor dtype. `float32` is recommended for GPU work. |
| `layout` | `Literal["CHW", "HWC"]` | `CHW` | Channel ordering. PyTorch convention is `CHW`. |

### Scheduler (fast mode only)

Ignored when `pipeline.mode = simple`.

```yaml
preprocessor:
  dask_scheduler: threads       # threads, processes, or single-threaded
  num_workers: 0
  chunk_size: 16
```

| Field | Type | Default | Description |
|---|---|---|---|
| `dask_scheduler` | `Literal["threads", "processes", "single-threaded"]` | `threads` | Dask scheduler type. `threads` is appropriate for most NumPy-based filters. |
| `num_workers` | `int >= 0` | `0` | Worker count. `0` auto-detects as `cpu_count() - 1`. |
| `chunk_size` | `int >= 1` | `16` | Frames per Dask task. Smaller values improve load balance; larger values reduce overhead. |

---

## ML Bridge Configuration

Configures Phase C: how processed frames become PyTorch batches.

### Common fields

```yaml
ml_bridge:
  task: classification          # or "super_resolution"
  batch_size: 32
  prefetch_factor: 4
  device: cpu
  pin_memory: true
  drop_last: false
```

| Field | Type | Default | Description |
|---|---|---|---|
| `task` | `Literal["classification", "super_resolution"]` | required | Task adapter to use. |
| `batch_size` | `int >= 1` | `32` | Frames per batch. Constrained by GPU memory. |
| `prefetch_factor` | `int >= 1` | `4` | Batches buffered ahead of consumption. See [modules/ml_bridge.md](modules/ml_bridge.md#prefetching-and-gpu-utilization). |
| `device` | `Literal["cpu", "cuda", "cuda:0", "cuda:1", ...]` | `cpu` | Where the output tensor lives. `cpu` is recommended; the consuming model can move to GPU. |
| `pin_memory` | `bool` | `true` | Pin memory for faster host-to-device transfer. Has no effect when `device = cpu` and the consumer is also on CPU. |
| `drop_last` | `bool` | `false` | Discard the final partial batch. Recommended `true` for training to keep batch statistics consistent. |

### Classification task

```yaml
ml_bridge:
  task: classification
  classification:
    label_dtype: int64
    num_classes: 3
```

| Field | Type | Default | Description |
|---|---|---|---|
| `label_dtype` | `Literal["int32", "int64"]` | `int64` | Dtype of the label tensor. PyTorch loss functions usually expect `int64`. |
| `num_classes` | `int >= 2` | required | Number of distinct classes. Used for validation; labels outside `[0, num_classes - 1]` cause errors. |

### Super-resolution task

```yaml
ml_bridge:
  task: super_resolution
  super_resolution:
    downsample_factor: 2
    downsample_method: bicubic
```

| Field | Type | Default | Description |
|---|---|---|---|
| `downsample_factor` | `int >= 2` | `2` | Linear downsampling factor for generating low-resolution inputs. Factor `2` gives `(H/2, W/2)`. |
| `downsample_method` | `Literal["bicubic", "bilinear", "nearest"]` | `bicubic` | Interpolation method. `bicubic` preserves more detail and is the standard choice for SR. |

---

## Logging and Error Bus

Optional section. Defaults are sensible for most use cases.

```yaml
logging:
  level: INFO                   # DEBUG, INFO, WARNING, ERROR
  format: json                  # json or text
  output: stdout                # stdout, stderr, or a file path

  error_bus:
    enabled: true
    output: ./logs/errors.jsonl
    include_full_provenance: true
```

| Field | Type | Default | Description |
|---|---|---|---|
| `level` | `Literal["DEBUG", "INFO", "WARNING", "ERROR"]` | `INFO` | Standard Python logging level. |
| `format` | `Literal["json", "text"]` | `json` | Output format. `json` is machine-parseable; `text` is human-readable. |
| `output` | `Literal["stdout", "stderr"]` or `path` | `stdout` | Destination for log output. |
| `error_bus.enabled` | `bool` | `true` | If `false`, dropped frames are not logged structurally (still appear in normal logs). |
| `error_bus.output` | `path` | `./logs/errors.jsonl` | JSONL file capturing all dropped-frame events. One event per line. |
| `error_bus.include_full_provenance` | `bool` | `true` | If `true`, each error event carries the full `Provenance` record. Disabling reduces log size at the cost of debuggability. |

---

## Environment Variable Substitution

Configuration values may reference environment variables using `${VAR_NAME}` syntax. Substitution happens before YAML parsing, so variables can appear in any value position.

```yaml
source:
  type: GenericDirectorySource
  root_path: ${DATA_ROOT}/deeplense_classification

ml_bridge:
  device: ${FLUX_DEVICE:-cpu}   # default to "cpu" if FLUX_DEVICE is unset
```

The default-value syntax `${VAR_NAME:-default}` provides a fallback when the variable is unset. Without a default, an unset variable causes a configuration error.

This mechanism is the recommended way to handle credentials, paths that vary between machines, and deployment-specific overrides without committing them to version control.

---

## Validation Rules

Beyond the per-field type and range constraints, the configuration validator enforces the following cross-field rules:

**Source-specific schema.** When `source.type = MockButlerSource`, fields like `manifest_path` are required; when `source.type = GenericDirectorySource`, they are forbidden. The schema is selected by the `type` field.

**Filter ordering compatibility.** The `filters` list may contain at most one instance of each filter type. Multiple instances of the same filter (e.g., two `GaussianFilter` entries) are rejected because the canonical ordering does not define how to schedule them.

**PSF compatibility.** When `RichardsonLucyDeconvolver` is configured, the PSF file at `psf_path` must exist and be loadable. This is checked at configuration time, not at first use.

**Normalization fixed values.** When `normalization.stats_source = fixed`, both `fixed_mean` and `fixed_std` must be provided. For `PerChannelZScore`, they must be lists of the same length as the number of channels in the data.

**Mode-specific fields.** When `pipeline.mode = simple`, the `dask_scheduler`, `num_workers`, and `chunk_size` fields under `preprocessor` are accepted but ignored, with a warning. This is to allow the same configuration to run in both modes.

**Task-adapter compatibility.** When `ml_bridge.task = classification` and the source does not provide labels (e.g., `GenericDirectorySource` with `label_from_subfolder: false`), validation fails at configuration time.

---

## Complete Examples

### Minimal classification pipeline

The smallest viable configuration. Reads from a labeled directory, applies basic preprocessing, emits classification batches.

```yaml
pipeline:
  mode: simple

source:
  type: GenericDirectorySource
  root_path: ./data/deeplense_classification
  extensions: [".npy"]
  label_from_subfolder: true

preprocessor:
  filters:
    - type: GaussianFilter
      sigma: 0.8
  normalization:
    type: PerChannelZScore
  tensor:
    dtype: float32
    layout: CHW

ml_bridge:
  task: classification
  batch_size: 32
  classification:
    num_classes: 3
```

### Production-quality classification pipeline

Full preprocessing chain, fast mode with Dask, structured error logging.

```yaml
pipeline:
  mode: fast
  queue_a_size: 128
  queue_b_size: 64

source:
  type: MockButlerSource
  collection: DP0_mock
  filter_bands: [g, r, i]
  tract: 4225
  patch: "3,4"
  manifest_path: ${DATA_ROOT}/mock_butler/manifest.yaml
  max_concurrent_fetches: 16
  max_retries: 3

preprocessor:
  cutout:
    enabled: true
    size: [64, 64]
    center_strategy: peak
  filters:
    - type: SigmaClipFilter
      sigma_threshold: 3.0
      iterations: 3
    - type: MedianFilter
      kernel_size: 3
    - type: RichardsonLucyDeconvolver
      psf_path: ${DATA_ROOT}/calibration/psf_r_band.fits
      iterations: 30
      regularization: tv
      regularization_weight: 0.01
    - type: GaussianFilter
      sigma: 0.6
    - type: WaveletDenoiser
      wavelet: db4
      level: 3
      threshold_method: bayes
  channel_handling:
    strategy: stack
    selected_bands: [g, r, i]
  normalization:
    type: PerChannelZScore
    stats_source: dataset
  tensor:
    dtype: float32
    layout: CHW
  dask_scheduler: threads
  num_workers: 0
  chunk_size: 16

ml_bridge:
  task: classification
  batch_size: 32
  prefetch_factor: 4
  device: cpu
  pin_memory: true
  drop_last: true
  classification:
    label_dtype: int64
    num_classes: 3

logging:
  level: INFO
  format: json
  output: ./logs/pipeline.log
  error_bus:
    enabled: true
    output: ./logs/errors.jsonl
```

### Super-resolution pipeline

Same data flow, different task adapter. Note the smaller batch size (SR tensors are larger) and the SR-specific configuration block.

```yaml
pipeline:
  mode: fast

source:
  type: MockButlerSource
  collection: DP0_mock
  filter_bands: [r]
  tract: 4225
  patch: "3,4"
  manifest_path: ${DATA_ROOT}/mock_butler/manifest.yaml

preprocessor:
  cutout:
    enabled: true
    size: [128, 128]
  filters:
    - type: SigmaClipFilter
      sigma_threshold: 3.0
    - type: MedianFilter
      kernel_size: 3
  channel_handling:
    strategy: stack
    selected_bands: [r]
  normalization:
    type: ZScore
    stats_source: dataset
  tensor:
    dtype: float32

ml_bridge:
  task: super_resolution
  batch_size: 16
  prefetch_factor: 4
  super_resolution:
    downsample_factor: 2
    downsample_method: bicubic
```

---

## See also

- [GETTING_STARTED.md](../GETTING_STARTED.md) — guided first run with a minimal configuration
- [ARCHITECTURE.md](ARCHITECTURE.md) — how configuration maps to runtime structure
- [DESIGN_PHILOSOPHY.md](DESIGN_PHILOSOPHY.md#validation-at-the-boundary) — why validation happens at load time
- [modules/extractor.md](modules/extractor.md), [modules/preprocessor.md](modules/preprocessor.md), [modules/ml_bridge.md](modules/ml_bridge.md) — phase-specific specifications