# Roadmap

This document specifies the planned evolution of FLUX. It serves three purposes: communicating direction to potential users and contributors, providing a stable reference for prioritization decisions, and articulating the long-term vision that justifies the architectural choices made today.

The roadmap is organized in three horizons: **current work** (the active milestone), **near-term** (the next two to three milestones, planned in detail), and **long-term** (directional, subject to revision as the project matures).

For the technical specification of FLUX as it exists today, see [ARCHITECTURE.md](docs/ARCHITECTURE.md). For the principles guiding what gets built and what does not, see [DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md).

---

## Table of Contents

1. [Vision](#vision)
2. [Current Milestone — v0.1](#current-milestone--v01)
3. [Near-Term — v0.2 and v0.3](#near-term--v02-and-v03)
4. [Long-Term — v1.0 and Beyond](#long-term--v10-and-beyond)
5. [Beyond the Pipeline — Web Interface and Backend](#beyond-the-pipeline--web-interface-and-backend)
6. [Domain Expansion Beyond DeepLense](#domain-expansion-beyond-deeplense)
7. [Out of Scope](#out-of-scope)
8. [How Plans Change](#how-plans-change)

---

## Vision

FLUX is being built to become a competitive, general-purpose data pipeline for machine learning in the astronomical and broader scientific domain. The initial development is anchored in DeepLense and gravitational lens detection, because that domain provides a concrete, well-defined, and scientifically valuable target — and because solving the data-pipeline problem there directly addresses gaps identified in prior ML4SCI work (specifically the limitations of RIPPLe and the failures observed in the 2023 Model IV experiment).

But the architecture is intentionally domain-agnostic. The three-phase decomposition, the plugin registries for sources, transforms, and task adapters, the streaming-over-materialization principle, the YAML configuration with Pydantic validation — none of these are specific to gravitational lensing. They are the general shape that any production-grade scientific ML pipeline ought to have.

The goal of FLUX is therefore twofold. In the short term, it is to deliver a robust, well-documented, well-tested pipeline that solves the DeepLense data-feeding problem at scale. In the long term, it is to become a tool that other scientific organizations — radio astronomy projects, exoplanet detection collaborations, time-domain survey teams, and beyond — can adopt, extend, and trust as a foundation for their own ML workflows.

This vision shapes every decision documented in this roadmap. Features that lock FLUX into one domain are deprioritized; features that strengthen its applicability across scientific domains are accelerated.

---

## Current Milestone — v0.1

**Status:** in active development.

**Theme:** establish the core pipeline and validate it on the DeepLense classification task.

The v0.1 milestone delivers the minimum viable FLUX. By the end of v0.1, a user can take a directory of DeepLense classification images, write a YAML configuration, and produce a PyTorch DataLoader that feeds a ResNet18 training loop without modification. Both `simple` and `fast` execution modes are functional. The five preprocessing filters are implemented with full test coverage. The MockButlerSource and GenericDirectorySource are functional. Documentation is complete (this milestone).

Concrete deliverables for v0.1:

- Three-phase asynchronous pipeline (`Pipeline`, `Extractor`, `Preprocessor`, `MLBridge`)
- `simple` and `fast` execution modes, both functional and producing equivalent output
- Five filter implementations: Gaussian, Median, SigmaClip, Wavelet, Richardson-Lucy
- Three normalization strategies: ZScore, MinMax, PerChannelZScore (with Welford's online algorithm)
- Two data sources: MockButlerSource, GenericDirectorySource
- Two task adapters: ClassificationAdapter, SuperResolutionAdapter
- Complete YAML configuration system with Pydantic validation
- Structured error bus with JSONL output
- Test suite covering >85% of the codebase, with module-specific targets per [CONTRIBUTING.md](CONTRIBUTING.md#testing-requirements)
- Full documentation set (the 15 documents enumerated in [README.md](README.md))
- An end-to-end demonstration notebook reproducing the DeepLense Common Test I results using FLUX

The exit criterion for v0.1 is that a fresh contributor, following [GETTING_STARTED.md](GETTING_STARTED.md), can have a working pipeline producing batches in under thirty minutes from `pip install`.

---

## Near-Term — v0.2 and v0.3

The near-term plan covers the two milestones following v0.1. They are planned in enough detail that a contributor can pick up specific tasks today, but the exact ordering and scope may shift in response to feedback from v0.1 users.

### v0.2 — Real Data Sources and Inference Workflows

**Theme:** move beyond mocked data and labeled training, supporting real LSST access and unlabeled inference.

The v0.1 pipeline operates on simulated data and labeled directories. v0.2 adds the components needed to consume real observational data and to support inference workflows where labels are not available.

Planned deliverables:

- `LSSTButlerSource` — production-grade implementation of `DataSource` against the real `lsst.daf.butler` API. Reuses the `ButlerQuery` schema introduced in v0.1 by `MockButlerSource`, so user configurations transfer between the two with only the `type` field changing.
- `PyVOSource` — implementation against the IVOA Virtual Observatory protocols, enabling FLUX to consume catalog-driven queries against any VO-compliant archive (NOIRLab, ESO, MAST, and others).
- `InferenceAdapter` — task adapter for unlabeled inference workflows. Emits `(image,)` tuples instead of `(image, label)`, allowing trained models to be applied to new survey data without requiring synthetic labels.
- A migration guide from RIPPLe configurations to FLUX configurations, lowering the barrier for existing RIPPLe users.
- A second demonstration notebook reproducing a super-resolution workflow on the DeepLense SR dataset.

### v0.3 — Performance Validation and Operational Maturity

**Theme:** measure, document, and tune performance to confirm the architectural claims.

The v0.1 and v0.2 milestones build the pipeline; v0.3 proves it. The central deliverable is the comprehensive set of benchmarks specified in [PERFORMANCE.md](docs/PERFORMANCE.md), populated with measured numbers from the three reference hardware tiers.

Planned deliverables:

- Full population of the benchmark tables in [PERFORMANCE.md](docs/PERFORMANCE.md) with measured numbers, across all three hardware tiers and all reference datasets
- Head-to-head comparison with RIPPLe under controlled conditions, with measurements published as the empirical justification for the architectural decisions in FLUX
- Tuning guide updates based on measured behavior rather than theoretical expectations
- Continuous benchmark CI: a subset of benchmarks runs on every release candidate, with regression detection against previous versions
- Optional Prefect integration for users who want visual pipeline monitoring and orchestration on top of FLUX
- Hardening of error recovery: chaos-testing the pipeline by deliberately injecting corrupted frames, network failures, and OOM conditions, and verifying that the error bus records every incident with full provenance

The exit criterion for v0.3 is that FLUX is operationally usable for a long-running scientific workflow, with documented performance characteristics and proven fault tolerance.

---

## Long-Term — v1.0 and Beyond

The long-term horizon is directional. The specific deliverables here are subject to revision based on what is learned from v0.1, v0.2, and v0.3.

### v1.0 — Public Release and Domain Expansion

**Theme:** stabilize the public API, broaden the supported domain, and announce general availability.

By v1.0, FLUX commits to API stability under semantic versioning. The configuration schema, the public class signatures, and the registered plugin names become a contract that the project will not break without a major version bump.

Anticipated deliverables:

- API stability guarantee under semver: breaking changes only at major version transitions, with deprecation warnings at least one minor version before removal
- Expanded plugin set: at minimum one source for time-domain surveys (variable star detection, transient classification), one source for radio astronomy data (HDF5-based formats), and corresponding domain-specific filters
- Adoption case studies: documented examples of FLUX in use on at least two scientific domains beyond DeepLense, validating the cross-domain claim made in [Vision](#vision)
- Comprehensive tutorial collection covering classification, super-resolution, inference at scale, and domain adaptation
- Conference presentation or workshop submission describing the architecture and lessons learned

### Beyond v1.0 — Possible Directions

These are research and engineering directions that have been considered but are not yet committed. They will be evaluated based on user feedback, the state of the surrounding ecosystem, and the contributor base at the time.

**Distributed pipelines.** Native support for spanning multiple machines via Dask Distributed or Ray. The architectural foundation already supports this — the queue-based decomposition is amenable to distributed scheduling — but the operational complexity warrants careful design and a dedicated RFC.

**GPU-accelerated preprocessing.** Several filters (Gaussian, FFT-based deconvolution) have natural GPU implementations via CuPy or PyTorch. A `Transform` subclass with GPU support would significantly accelerate the preprocessing-bound regime where Phase B is the bottleneck.

**Real-time streaming sources.** Adapters for live telescope data feeds (Kafka-based alert streams from LSST, ZTF, or future observatories), enabling FLUX to operate not only on archived data but on the firehose of incoming observations.

**Active learning loops.** Integration with model uncertainty estimates to feed the most informative frames back into the pipeline preferentially. This requires bidirectional communication between the consumer model and FLUX, which the current pure-streaming model does not support — a careful architectural extension.

**Pipeline composition.** Support for chaining multiple FLUX pipelines (e.g., a coarse classification feeding a finer-grained downstream task) with shared caching and intermediate persistence.

Each of these requires its own RFC before implementation begins. None are committed yet.

---

## Beyond the Pipeline — Web Interface and Backend

A natural evolution of FLUX, planned for after the core pipeline reaches v1.0, is a web-based demonstration and exploration interface. The motivation is twofold: providing scientists with a way to experiment with FLUX configurations without writing code, and providing a tangible, demonstrable artifact that communicates the project's capabilities to a broader audience.

The planned interface follows a standard separation of concerns:

- **Frontend** — a React application that presents an interactive workspace where users upload an image (or select one from a curated dataset), choose a preprocessing configuration via form-based controls, run inference against a pre-loaded DeepLense model, and visualize the results. The headline feature is **live Grad-CAM visualization**, showing which regions of the image the model relied on for its prediction. The frontend is a thin client; all computation happens on the backend.

- **Backend** — a FastAPI service exposing FLUX functionality through a REST API. Endpoints include configuration validation, single-image inference, and Grad-CAM extraction. The backend wraps the same FLUX pipeline used in the Python API, so the web interface is a presentation layer rather than a parallel implementation. This guarantees that anything demonstrable in the web app is also achievable from the Python API, with the same results.

- **Deployment** — containerized via Docker, deployable on standard cloud platforms or on-premises. The backend is stateless; the frontend is static. This deployment model is appropriate for both public demonstrations and for institutional internal use, where a research group might want to make FLUX accessible to non-developer scientists.

The web interface is explicitly a v1.x deliverable. It is not part of v0.1, v0.2, or v0.3, and committing to it before the core pipeline is mature would distort priorities. But it is part of the long-term plan and is what shapes some of the architectural decisions today: the structured error bus uses JSON because it will eventually be consumed by a frontend; the configuration schema uses Pydantic because Pydantic also generates JSON Schema, which can drive a form-based UI directly.

---

## Domain Expansion Beyond DeepLense

The project is initially developed in the context of ML4SCI's DeepLense, and its first complete demonstration reproduces a DeepLense classification task. This is a deliberate choice: building against a concrete, scientifically valuable target produces a more focused and useful tool than starting from a generic abstraction.

But the architecture has no DeepLense-specific assumptions. The three-phase decomposition, the plugin registries, the streaming model, the YAML configuration — these are generic. Other organizations and research groups working on machine learning over astronomical or scientific imaging are explicitly invited to adopt, extend, and contribute to FLUX:

- **Other gravitational lensing efforts** — collaborations beyond ML4SCI working on lensing detection or analysis can use FLUX directly with minimal configuration changes.
- **Radio astronomy** — projects working with SKA, VLA, or ALMA data can implement a custom `DataSource` and reuse the entire preprocessing and ML Bridge infrastructure.
- **Time-domain surveys** — variable star catalogs, transient detection pipelines, and supernova classification efforts share the same fundamental data pattern (image streams to ML models) that FLUX is designed to handle.
- **Exoplanet detection** — light-curve preprocessing has different specifics from imaging but the architectural pattern (asynchronous pipeline, configurable transforms, ML-ready output) maps cleanly.
- **Non-astronomy scientific imaging** — biomedical microscopy, materials science, or remote sensing applications that share the production-ML-pipeline shape.

The barrier to adopting FLUX in a new domain is intentionally low: implement a `DataSource` for the domain's data format, optionally implement domain-specific filters as `Transform` subclasses, and reuse everything else. Every plugin authored for a new domain enriches FLUX's plugin ecosystem and makes the next domain easier to onboard.

Cross-domain adoption is not a side effect of FLUX — it is a goal. Pull requests adding sources, filters, or adapters for new domains are welcome and are reviewed with the same care as DeepLense-specific contributions. Documentation contributions describing how FLUX is used in a new domain are encouraged and will be linked from the project's main documentation.

---

## Out of Scope

The roadmap is constrained by what FLUX is — and is not — willing to become. The following are explicitly out of scope, regardless of demand:

**Model training and architecture research.** FLUX produces DataLoaders. It does not contain training loops, loss functions, optimizers, or model architectures. Those belong to DeepLense and equivalent projects. This boundary is what allows FLUX to remain useful across many different model libraries and research efforts.

**General-purpose image processing.** FLUX is not OpenCV, scikit-image, or Astropy. The five filters shipped with FLUX are specifically chosen for the noise characteristics of telescope imaging in the ML preprocessing context. Image processing for other purposes — visualization, photometry, scientific measurement — is outside the project's scope.

**Cluster orchestration.** FLUX runs on one machine. Spanning multiple machines requires layering Dask Distributed, Ray, or a workload manager (Slurm, Kubernetes) on top of FLUX. The pipeline is designed to be a building block in larger orchestration systems, not to be one itself.

**Data archival or storage.** FLUX does not provide a storage backend. It consumes from sources and produces ephemeral batches. Long-term storage, cataloging, and metadata management are responsibilities of upstream tools like Butler or downstream tools like the user's own database.

**A graphical configuration editor.** The web interface described above is for inference and visualization, not for editing complex YAML configurations. Configuration is a code-like artifact that should be version-controlled and reviewed; turning it into a clickable UI obscures decisions that scientists should be making explicitly.

These exclusions are what allow FLUX to do its actual job well. Each one represents a genuine demand that someone, eventually, will make of the project, and each one will be declined with a pointer to this section.

---

## How Plans Change

Plans on this roadmap are commitments at the level of intent, not guarantees of delivery. The following rules govern how the roadmap evolves:

**Current milestone is firm.** Once a milestone is in active development, its scope is fixed. New work proposed during the milestone is deferred to a future milestone unless it is critical to the current one.

**Near-term milestones are negotiable.** v0.2 and v0.3 plans may be reordered, expanded, or trimmed based on what is learned from earlier work. Significant changes are documented as updates to this file with an entry in the changelog.

**Long-term plans are directional.** Items in the v1.0 and beyond sections represent the project's intent at the time of writing. They are subject to revision as the surrounding ecosystem evolves and as the contributor base shapes priorities. A change to long-term plans does not require an RFC — but a substantive revision of this document does, since the roadmap itself is governance content.

**The vision is stable.** The fundamental purpose of FLUX — a competitive, domain-agnostic data pipeline for scientific ML, initially anchored in DeepLense — is not negotiable on the timescale of this document. A change to the vision would require a new project, not an amendment to this one.

---

## See also

- [README.md](README.md) — project overview and current state
- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — what the current pipeline looks like
- [DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md) — the principles that constrain what FLUX becomes
- [RFC_PROCESS.md](docs/RFC_PROCESS.md) — how significant roadmap items become concrete proposals
- [CONTRIBUTING.md](CONTRIBUTING.md) — how to participate in advancing the roadmap