# Design Philosophy

This document explains the **why** behind FLUX's architecture. [ARCHITECTURE.md](ARCHITECTURE.md) describes what the system is; this document describes what it tries to be, what trade-offs it accepts, and what it deliberately refuses to be.

It is intended for two audiences: contributors deciding how to extend FLUX without violating its principles, and reviewers evaluating the technical maturity of the project.

---

## Table of Contents

1. [Core Principles](#core-principles)
2. [Asynchrony over Threading](#asynchrony-over-threading)
3. [Decoupling over Cleverness](#decoupling-over-cleverness)
4. [Validation at the Boundary](#validation-at-the-boundary)
5. [Two Modes, One Code Path](#two-modes-one-code-path)
6. [Streaming over Materialization](#streaming-over-materialization)
7. [Fail Loud, Recover Locally](#fail-loud-recover-locally)
8. [What FLUX is Not](#what-flux-is-not)
9. [SOLID Applied](#solid-applied)

---

## Core Principles

FLUX is guided by five principles, in priority order. When two principles conflict, the higher-priority one wins.

1. **Correctness over speed.** A pipeline that produces wrong results faster is worse than a pipeline that produces right results slowly. Every optimization must preserve scientific correctness.
2. **Maintainability over elegance.** Code that a working scientist can read and modify wins over code that is theoretically beautiful but requires distributed-systems expertise to understand.
3. **Predictability over peak performance.** Bounded memory and consistent throughput beat occasional bursts of speed followed by stalls.
4. **Composition over configuration.** Behavior is changed by composing small components, not by adding flags to a monolithic class.
5. **Explicit over implicit.** Magic is forbidden. Every transformation is visible in the YAML configuration and in the structured log output.

These are not abstract values — every concrete decision in [ARCHITECTURE.md](ARCHITECTURE.md) traces back to one of them.

---

## Asynchrony over Threading

FLUX's concurrency model is built on `asyncio`, not on threads or multiprocessing. This is a deliberate choice.

**Threads are wrong for this workload** because Python's Global Interpreter Lock prevents true CPU parallelism within a single process, and our bottleneck is mostly I/O (data fetching) plus CPU-bound numerical work (preprocessing). The I/O part benefits from concurrency without parallelism — exactly what `asyncio` provides.

**Multiprocessing is wrong as a default** because it imposes serialization overhead on every `RawFrame` crossing process boundaries, and the data being passed is large (image arrays). For a single-machine pipeline, the cost of pickling tensors between processes erases any gains.

**The hybrid solution:** `asyncio` orchestrates the three phases. Within Phase B, when CPU parallelism is needed, **Dask delayed** is used because Dask handles serialization intelligently and supports both threads and processes via its scheduler — and crucially, lets the user choose without changing application code. This gives FLUX the right concurrency primitive at each layer.

The result: Phase A is bottlenecked by network I/O and benefits from `asyncio`. Phase B is bottlenecked by CPU and benefits from Dask. Phase C is bottlenecked by GPU and benefits from prefetching. Each phase uses the right tool for its actual constraint.

---

## Decoupling over Cleverness

A common temptation in pipeline design is to "optimize" by sharing state across stages — the Preprocessor reaches into the Extractor to peek at the next file, the ML Bridge tells the Preprocessor to skip a filter. These optimizations always pay off in the short term and always cause maintenance disasters in the long term.

FLUX refuses this temptation by enforcing a strict rule: **phases communicate only through the queue interface**. There is no `pipeline.preprocessor.set_filter_for_next_frame()`. There is no shared global state. If Phase B needs information from Phase A, it must arrive in the `RawFrame` metadata, full stop.

This has three consequences:

- Each phase is **independently testable**. Unit tests for the Preprocessor inject `RawFrame` objects directly without spinning up the Extractor. This is why FLUX targets >85% test coverage as a project goal.
- Each phase is **independently replaceable**. A future contributor wanting to swap the Extractor for one that talks to a different telescope changes only Phase A and leaves the rest untouched.
- Each phase is **independently profileable**. Bottlenecks are isolated. If `fast` mode underperforms, the metrics tell us exactly which phase is slow.

The architectural cost is some duplication — metadata that travels through all three phases is carried forward redundantly. The maintenance benefit dwarfs the cost.

---

## Validation at the Boundary

Configuration errors are the single largest source of silent bugs in scientific software. A typo in a YAML file does not crash the pipeline — it just produces wrong results, which the scientist trusts because "the code ran without errors."

FLUX rejects this failure mode by validating **all** configuration through Pydantic models at the very moment the YAML is loaded, before a single byte of data is fetched. If a filter is configured with `kernel_size: -3`, the pipeline refuses to start and tells the user exactly which line of which file has the problem.

This is an application of the principle: **errors are cheaper at the boundary than in the middle**. A configuration error caught at startup costs the user 30 seconds. The same error caught after 6 hours of processing costs the user 6 hours of compute and a stale paper deadline.

The Pydantic schemas live in `flux/config/schema.py` and are the single source of truth for the configuration surface. They are also what generates the human-readable schema reference in [CONFIGURATION.md](CONFIGURATION.md) — the documentation cannot drift from the code because both are derived from the same Pydantic models.

---

## Two Modes, One Code Path

The `simple` and `fast` execution modes described in [ARCHITECTURE.md](ARCHITECTURE.md) might naively suggest two separate implementations. They are not.

The same `Pipeline` class implements both. The only difference between modes is the **scheduler** the orchestrator uses. In `simple` mode, the scheduler is a sequential walker: extract → preprocess → emit batch, repeat. In `fast` mode, the scheduler is `asyncio.gather` over three concurrent tasks. The phase implementations themselves are identical.

This is the **maintainability contract** stated in [ARCHITECTURE.md](ARCHITECTURE.md): a bug fix in the Preprocessor benefits both modes. A new filter added to the registry is available in both modes. A scientist who develops in `simple` mode and deploys to `fast` mode does not need to re-test their preprocessing chain.

The cost of this discipline is that some optimizations possible in `fast` mode are not exploited (for example, we could pre-fuse certain filter operations only in async mode). We accept that cost. **A 1.3x speedup is not worth a 2x maintenance burden.**

---

## Streaming over Materialization

FLUX never assumes the dataset fits in memory. Phase A produces a stream. Phase B processes the stream item by item. Phase C exposes the stream as `IterableDataset`. At no point in the pipeline is the full dataset materialized.

This decision is forced by the target use case: LSST surveys are measured in petabytes. The pipeline must work on a 100-image test set and on a 100-million-image survey with the same code, the same configuration, and the same memory footprint.

The architectural consequence is that some operations that would be trivial on a fully materialized dataset (global statistics, sorting by quality, dataset-wide stratified splits) must be implemented as **streaming algorithms** with bounded memory — for example, normalization statistics use Welford's online algorithm rather than `np.mean`, and any "global" operation either operates on a configurable sample window or is explicitly two-pass.

This is harder, but it is not optional. Materialized pipelines do not scale, and unscalable pipelines are not pipelines — they are scripts.

---

## Fail Loud, Recover Locally

FLUX's failure model is a deliberate paradox: errors must be **loud** at the configuration and infrastructure level, and **silent** at the data level.

**Loud failures** apply to configuration errors, missing dependencies, network unavailability, and out-of-memory conditions. These are the user's responsibility to fix, and hiding them is hostile.

**Silent failures** apply to corrupted individual frames, transient I/O glitches, NaN values in specific images. These are facts of life when processing real telescope data — a one-in-a-million bad pixel must not stop a million-image pipeline. They are logged to the structured error bus and the pipeline moves on.

The line between the two categories is precise: **if fixing it requires user action, fail loud; if fixing it requires retrying or skipping, fail silent**. The error bus exists so that "silent" never means "invisible" — every dropped frame is recorded with full provenance, retrievable by the user after the fact.

This is a direct response to RIPPLe's failure mode (one bad image halts the entire pipeline) and to the 2023 DeepLense Model IV failure (corrupted preprocessing went unnoticed because there was no error bus to consult).

---

## What FLUX is Not

Equally important to what FLUX is. FLUX deliberately is **not**:

**Not a model trainer.** FLUX produces DataLoaders. It does not contain training loops, loss functions, or optimizers. Those belong in DeepLense. This boundary is non-negotiable — projects that conflate data pipelines with training code become unmaintainable.

**Not a distributed cluster manager.** FLUX runs on one machine. If you need to span 100 machines, use Dask Distributed or Ray on top of FLUX. The pipeline is designed to be embedded in a larger orchestration system, not to be one.

**Not a replacement for Butler or PyVO.** FLUX wraps these tools to make them ergonomic for ML, but it does not reimplement their query semantics. When in doubt, defer to the canonical APIs.

**Not a general-purpose image processing library.** The five filters shipped with FLUX are specifically chosen for astronomical image preprocessing in the context of gravitational lens detection. FLUX is not OpenCV. Users wanting unrelated transformations should write their own `Transform` subclass.

**Not a black box.** Every transformation, every filter, every default value has a documented mathematical or empirical justification in the [math/](math/) directory or the module specifications. If a contribution proposes "use this clever trick," the trick must come with a reference or a derivation.

These exclusions are what allow FLUX to do its actual job well.

---

## SOLID Applied

The five SOLID principles are concrete in FLUX.

**Single Responsibility.** Each phase has one responsibility — extraction, transformation, or assembly. Inside each phase, each filter has one responsibility — `GaussianFilter` does not also normalize. The `Transform` interface is one method (`apply`) by design.

**Open/Closed.** New data sources, filters, and task adapters are added by registering new classes in their respective registries. The pipeline core never changes. Adding support for a new dataset format does not require modifying `Pipeline.run()`.

**Liskov Substitution.** Any `DataSource` implementation must be substitutable for `MockButlerSource` without the Preprocessor noticing. The interface contract (`async fetch() -> AsyncIterator[RawFrame]`) is enforced by the type system and tested with abstract base class tests applied to every concrete implementation.

**Interface Segregation.** The three abstract interfaces (`DataSource`, `Transform`, `TaskAdapter`) are minimal — each requires only the methods strictly necessary. There is no `BaseEverything` god-class that forces implementers to stub out unused methods.

**Dependency Inversion.** The `Pipeline` orchestrator depends on the three abstract interfaces, not on concrete implementations. `MockButlerSource` is wired in by the configuration loader, not by the orchestrator. This means the entire pipeline can be tested with stub implementations injected from a test fixture.

---

## See also

- [ARCHITECTURE.md](ARCHITECTURE.md) — the concrete realization of these principles
- [RFC_PROCESS.md](RFC_PROCESS.md) — how to propose changes that respect these principles
- [CONTRIBUTING.md](../CONTRIBUTING.md) — how the principles are enforced in pull requests