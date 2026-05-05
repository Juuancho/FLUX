# FLUX

> **F**ast **L**SST-to-DeepLense **U**nified e**X**tractor — an asynchronous, fault-tolerant data pipeline connecting astronomical surveys with the DeepLense machine learning ecosystem.

[![Python](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/)
[![PyTorch](https://img.shields.io/badge/PyTorch-2.0+-EE4C2C.svg)](https://pytorch.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Documentation](https://img.shields.io/badge/docs-mkdocs-informational.svg)](docs/)
[![Code style](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)

---

## What is FLUX?

FLUX is a production-grade data pipeline that bridges raw astronomical observations and the [DeepLense](https://github.com/ML4SCI/DeepLense) machine learning models for gravitational lens detection and analysis.

It exists to solve a specific problem: DeepLense's models achieve excellent results on simulated data, but operating them on real telescope observations requires a robust, scalable data pipeline that does not exist as a standalone, reusable component. Previous attempts (RIPPLe, 2025) established a working connection but used a synchronous, monolithic architecture that creates a critical bottleneck at survey scale — the GPU sits idle while data is fetched and preprocessed sequentially.

FLUX redesigns that pipeline as three independent asynchronous stages connected by queues. Slow I/O never blocks preprocessing. A corrupted image never stops the pipeline. The GPU receives a continuous stream of preprocessed batches, never waiting.

## Why this matters

The Vera Rubin Observatory's LSST survey will produce approximately 20 terabytes of data per night for ten years. Strong gravitational lenses — among the rarest and most scientifically valuable objects in the universe — must be discovered within that flood. Operating ML models at that scale is fundamentally an engineering problem, not a modeling problem.

FLUX provides the engineering layer.

## Key features

- **Decoupled three-phase architecture** — Extractor, Preprocessor, and ML Bridge run as independent asynchronous stages
- **Two execution modes** — `simple` for sequential operation (single command, no distributed knowledge required), `fast` for asynchronous operation with Dask parallelism
- **Five astronomical preprocessing filters** — Gaussian, Median, Sigma clipping, Wavelet denoising, and Richardson-Lucy PSF deconvolution
- **Pluggable data sources** — official LSST Butler mock interface and a generic adapter for any astronomical dataset (HuggingFace, custom directories, etc.)
- **Native DeepLense integration** — drop-in PyTorch `IterableDataset` compatible with classification and super-resolution models
- **Fault-tolerant by design** — automatic retries, structured error logging, no single point of failure
- **YAML-driven configuration** — scientists modify behavior without touching source code, with full Pydantic validation

## Quick start

```bash
pip install fluxAstroML
```

```python
from fluxAstroML import Pipeline

pipeline = Pipeline.from_config("configs/classification_demo.yaml")
results = pipeline.run()
```

Full installation instructions and a guided first run are available in [GETTING_STARTED.md](GETTING_STARTED.md).

## Architecture overview

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Phase A       │    │     Phase B      │    │    Phase C      │
│   Extractor     │───▶│  Preprocessor    │───▶│   ML Bridge     │
│ (Butler / Mock) │    │ (Filters + Norm) │    │ (DeepLense I/O) │
└─────────────────┘    └──────────────────┘    └─────────────────┘
        │                       │                       │
        └───────────── Async queues + retries ──────────┘
```

Each phase runs independently. A failure in one does not stall the others.
Detailed design rationale is in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md).

## Project structure

```
flux/
├── README.md
├── GETTING_STARTED.md
├── CONTRIBUTING.md
├── ROADMAP.md
├── .gitignore
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DESIGN_PHILOSOPHY.md
│   ├── RFC_PROCESS.md
│   ├── CONFIGURATION.md
│   ├── PERFORMANCE.md
│   ├── modules/
│   │   ├── extractor.md
│   │   ├── preprocessor.md
│   │   └── ml_bridge.md
│   └── math/
│       ├── filters.md
│       ├── normalization.md
│       └── psf_deconvolution.md
├── flux/                  # source code
├── configs/               # example YAML configurations
├── tests/                 # pytest suite
└── notebooks/             # demonstration notebooks
```

## Documentation map

| Audience | Read first |
|---|---|
| **First-time visitor** | [README.md](README.md) → [GETTING_STARTED.md](GETTING_STARTED.md) |
| **Scientist using the pipeline** | [GETTING_STARTED.md](GETTING_STARTED.md) → [docs/CONFIGURATION.md](docs/CONFIGURATION.md) |
| **Developer contributing code** | [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) → [docs/DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md) → [CONTRIBUTING.md](CONTRIBUTING.md) |
| **ML engineer integrating models** | [docs/modules/ml_bridge.md](docs/modules/ml_bridge.md) |
| **Researcher reviewing methodology** | [docs/math/](docs/math/) |
| **Operations / benchmarking** | [docs/PERFORMANCE.md](docs/PERFORMANCE.md) |

## Roadmap highlights

- **v0.1** — Core pipeline with mock Butler and DeepLense classification support
- **v0.2** — Generic dataset adapter, super-resolution model support
- **v0.3** — Full async execution with Dask, performance benchmarks
- **v1.0** — Interactive web demo (FastAPI + React) with live Grad-CAM visualization

Full roadmap in [ROADMAP.md](ROADMAP.md).

## Related work

This project builds directly on [RIPPLe (GSoC 2025)](https://github.com/ML4SCI/DeepLense/tree/main/DeepLense_Data_Processing_Pipeline_for_the_LSST) and addresses the limitations identified in the 2023 DeepLense Model IV experiment, which demonstrated that real observational data requires preprocessing strategies not available in the simulated-data workflow. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for a detailed comparison.

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) for the development workflow and [docs/RFC_PROCESS.md](docs/RFC_PROCESS.md) for proposing significant changes.

## License

MIT — see [LICENSE](LICENSE) for details.

## Author

**Juan José Caraballo Nieves**
Systems Engineering, Corporación Universitaria Rafael Núñez
[GitHub](https://github.com/Juuancho) · [LinkedIn](https://www.linkedin.com/in/juanjoox/)

---

*FLUX is an independent research and engineering project, not officially affiliated with ML4SCI, DeepLense, or the Rubin Observatory. It is built to be compatible with their open ecosystems.*