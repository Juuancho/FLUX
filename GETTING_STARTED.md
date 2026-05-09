# Getting Started

This guide walks through installing FLUX, preparing a small dataset, running a first pipeline, and consuming the output in a PyTorch training loop. It is intended for users with basic Python and PyTorch experience but no prior knowledge of FLUX, LSST, or asynchronous pipelines.

By the end of this guide, a working FLUX pipeline produces preprocessed batches consumable by any DeepLense model.

For the configuration reference, see [CONFIGURATION.md](docs/CONFIGURATION.md). For the architectural context, see [ARCHITECTURE.md](docs/ARCHITECTURE.md).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Preparing a Dataset](#preparing-a-dataset)
4. [Writing the First Configuration](#writing-the-first-configuration)
5. [Running the Pipeline](#running-the-pipeline)
6. [Consuming the Output in PyTorch](#consuming-the-output-in-pytorch)
7. [Switching to Fast Mode](#switching-to-fast-mode)
8. [Troubleshooting](#troubleshooting)
9. [Next Steps](#next-steps)

---

## Prerequisites

Before installing FLUX, the following must be available on the system:

- Python 3.10 or newer
- A working `pip` and access to PyPI
- At least 4 GB of free disk space for FLUX itself, its dependencies, and a small example dataset
- Optional: a CUDA-capable GPU with PyTorch's CUDA build installed. CPU-only operation is fully supported.

A virtual environment is strongly recommended. The examples below use `venv`, but `conda`, `uv`, or `poetry` work equally well.

```bash
python -m venv .venv
source .venv/bin/activate          # Linux / macOS
.venv\Scripts\activate             # Windows
```

---

## Installation

Install FLUX from PyPI:

```bash
pip install fluxAstroML
```

This installs FLUX along with its required dependencies: PyTorch, NumPy, SciPy, scikit-image, PyWavelets, Astropy, Pydantic, Dask, and PyYAML.

Verify the installation by importing the package and printing its version:

```bash
python -c "import flux; print(flux.__version__)"
```

To install the development version directly from the GitHub repository:

```bash
pip install git+https://github.com/Juuancho/flux.git
```

---

## Preparing a Dataset

FLUX consumes any directory of `.npy` or `.fits` files. For this guide, a small subset of the publicly available DeepLense classification dataset is sufficient.

The expected directory structure for `GenericDirectorySource` with `label_from_subfolder: true` is:

```
data/
└── deeplense_classification/
    ├── no/
    │   ├── image_0001.npy
    │   ├── image_0002.npy
    │   └── ...
    ├── sphere/
    │   ├── image_0001.npy
    │   └── ...
    └── vort/
        ├── image_0001.npy
        └── ...
```

Each `.npy` file contains a single image array. For the DeepLense classification dataset, arrays are shape `(150, 150)` with `float32` values normalized to `[0, 1]`. FLUX will repeat this single channel into three channels automatically when configured for ResNet-style consumption.

If the data is provided as a `.zip` or `.tar.gz` archive, extract it before pointing FLUX at the resulting directory.

---

## Writing the First Configuration

Create a file `configs/first_run.yaml` with the following content:

```yaml
pipeline:
  mode: simple

source:
  type: GenericDirectorySource
  root_path: ./data/deeplense_classification
  extensions: [".npy"]
  label_from_subfolder: true
  limit: 100                    # only the first 100 frames, to keep the run short

preprocessor:
  filters:
    - type: GaussianFilter
      sigma: 0.8
  normalization:
    type: PerChannelZScore
    stats_source: per_image
  tensor:
    dtype: float32
    layout: CHW

ml_bridge:
  task: classification
  batch_size: 16
  classification:
    num_classes: 3

logging:
  level: INFO
  format: text
  output: stdout
```

A few choices in this configuration are intentional for a first run:

- `mode: simple` runs all phases sequentially in a single process. No Dask, no asyncio internals to debug.
- `limit: 100` prevents the pipeline from processing the entire dataset on the first try.
- `stats_source: per_image` avoids the upfront cost of computing dataset-wide statistics.
- `format: text` produces human-readable log output instead of JSON.

For a full reference of every configuration field, see [CONFIGURATION.md](docs/CONFIGURATION.md).

---

## Running the Pipeline

The simplest way to run a pipeline is from a Python script. Create `run_pipeline.py`:

```python
from flux import Pipeline

pipeline = Pipeline.from_config("configs/first_run.yaml")
loader   = pipeline.as_dataloader()

for i, (images, labels) in enumerate(loader):
    print(f"Batch {i}: images={images.shape}, labels={labels.shape}")
    if i == 4:
        break
```

Run it:

```bash
python run_pipeline.py
```

Expected output:

```
[INFO] flux.config: loaded configuration from configs/first_run.yaml
[INFO] flux.pipeline: starting pipeline in 'simple' mode
[INFO] flux.extractor: GenericDirectorySource resolved 100 frames
[INFO] flux.preprocessor: applying 1 filter(s)
[INFO] flux.ml_bridge: classification adapter, batch_size=16
Batch 0: images=torch.Size([16, 3, 150, 150]), labels=torch.Size([16])
Batch 1: images=torch.Size([16, 3, 150, 150]), labels=torch.Size([16])
Batch 2: images=torch.Size([16, 3, 150, 150]), labels=torch.Size([16])
Batch 3: images=torch.Size([16, 3, 150, 150]), labels=torch.Size([16])
Batch 4: images=torch.Size([16, 3, 150, 150]), labels=torch.Size([16])
```

The pipeline produced five batches of 16 images each, totaling 80 frames out of the 100 fetched. The remaining 20 frames would be in a final partial batch (yielded if `drop_last: false`).

---

## Consuming the Output in PyTorch

The DataLoader produced by FLUX is a standard `torch.utils.data.DataLoader` instance. Any PyTorch training loop consumes it without modification.

Below is a minimal classification training loop that consumes the FLUX output:

```python
import torch
import torch.nn as nn
from torchvision.models import resnet18
from flux import Pipeline

# Load FLUX pipeline
pipeline = Pipeline.from_config("configs/first_run.yaml")
loader   = pipeline.as_dataloader()

# Set up a small ResNet18
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model  = resnet18(num_classes=3).to(device)
loss_fn = nn.CrossEntropyLoss()
optim   = torch.optim.AdamW(model.parameters(), lr=1e-4)

# One epoch over the FLUX-produced data
model.train()
for images, labels in loader:
    images, labels = images.to(device), labels.to(device)

    logits = model(images)
    loss   = loss_fn(logits, labels)

    optim.zero_grad()
    loss.backward()
    optim.step()

    print(f"loss = {loss.item():.4f}")
```

Three things to note:

`images.to(device)` is required because FLUX's default device is `cpu` (configurable via `ml_bridge.device`). Moving the tensor to GPU is the consumer's responsibility, which keeps FLUX itself GPU-agnostic.

The `DataLoader` is iterable but not subscriptable. `for batch in loader` works; `loader[0]` does not. This is a property of `IterableDataset`, intentional for streaming use cases.

The loop terminates cleanly when the source is exhausted. There is no explicit `len(loader)` because the upstream stream may have unknown length — design choice documented in [DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md#streaming-over-materialization).

---

## Switching to Fast Mode

Once the pipeline runs correctly in `simple` mode, switching to `fast` mode is a one-line configuration change:

```yaml
pipeline:
  mode: fast                    # was "simple"
```

No other change is needed. The same `Pipeline.from_config` call now spawns three concurrent asyncio tasks, the Preprocessor delegates to Dask for parallel filtering, and the ML Bridge prefetches batches ahead of consumption.

For a 100-frame dataset on a typical 4-core laptop, the speedup is modest (the overhead of starting Dask is comparable to the work). For a 100,000-frame dataset, expect 4–8x throughput improvement. Detailed benchmarks are in [PERFORMANCE.md](docs/PERFORMANCE.md).

If `fast` mode produces different results than `simple` mode, that is a bug. The maintainability contract of FLUX guarantees mode-equivalent output for the same configuration. Please report such cases via the GitHub issue tracker.

---

## Troubleshooting

### `ModuleNotFoundError: No module named 'flux'`

The package was not installed in the current Python environment. Verify the active environment with `which python` (Linux/macOS) or `where python` (Windows) and reinstall.

### `ConfigurationError: validation failed for source.root_path`

The path in the configuration does not exist or is not readable. Check the spelling and the working directory from which the script runs (relative paths are resolved against the current working directory, not against the configuration file's location).

### `MissingLabelError: frame has no label`

The source is configured without `label_from_subfolder: true`, but the ML Bridge is configured for classification. Either enable label inference from the directory structure or change the task to one that does not require labels (super-resolution, or `InferenceAdapter` once available in v0.2).

### `RuntimeError: stack expects each tensor to be equal size`

Frames in the dataset have inconsistent shapes. This typically means the cutout step is disabled and the source contains files of varying dimensions. Either enable cutout extraction in the preprocessor or pre-process the dataset to a consistent shape.

### Pipeline appears stuck in `fast` mode

In `fast` mode, the absence of progress messages does not necessarily mean a hang — the asyncio loop may simply be waiting for I/O. If the pipeline is genuinely stuck, switch to `simple` mode for diagnostics. Persistent issues in `fast` mode that do not appear in `simple` mode should be reported as bugs.

### High memory usage

Memory usage grows with `queue_a_size`, `queue_b_size`, `prefetch_factor`, and `batch_size`. For memory-constrained environments (such as the GTX 1650 with 4 GB VRAM), reduce these values. Sensible starting points: `queue_a_size: 32`, `queue_b_size: 16`, `prefetch_factor: 2`, `batch_size: 16`.

---

## Next Steps

With a working pipeline, the natural next steps are:

**Add real preprocessing.** Replace the single-filter chain in `first_run.yaml` with the full chain documented in [modules/preprocessor.md](docs/modules/preprocessor.md). The five filters together are what make FLUX appropriate for real telescope data.

**Switch to dataset statistics.** Change `stats_source: per_image` to `stats_source: dataset` for transfer learning workflows. Welford's online algorithm computes statistics in a single streaming pass.

**Try the Mock Butler source.** When LSST-style queries are needed, switch from `GenericDirectorySource` to `MockButlerSource`. See [modules/extractor.md](docs/modules/extractor.md) for query semantics.

**Train a real model.** The minimal training loop above can be extended into a full DeepLense training script. The key insight is that the FLUX-produced DataLoader is interchangeable with any standard PyTorch DataLoader.

**Read the design documents.** [ARCHITECTURE.md](docs/ARCHITECTURE.md) and [DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md) explain the reasoning behind FLUX's structure and what trade-offs were accepted.

**Contribute.** New filters, sources, and task adapters are added through the registry mechanism documented in [CONTRIBUTING.md](CONTRIBUTING.md). The contribution path for a new filter is approximately one Python file plus a corresponding test file.

---

## See also

- [README.md](README.md) — project overview
- [CONFIGURATION.md](docs/CONFIGURATION.md) — full YAML schema reference
- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — system architecture
- [PERFORMANCE.md](docs/PERFORMANCE.md) — benchmarks and tuning
- [CONTRIBUTING.md](CONTRIBUTING.md) — how to contribute