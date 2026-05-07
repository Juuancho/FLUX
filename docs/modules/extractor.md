# Module: Extractor (Phase A)

The Extractor is the first phase of the FLUX pipeline. Its job is to retrieve raw astronomical data from a configured source and emit individual frames into Queue A. This document is the authoritative specification of its public interface, internal structure, and operational behavior.

For the role of the Extractor in the broader pipeline, see [ARCHITECTURE.md](../ARCHITECTURE.md). For the principles guiding its design, see [DESIGN_PHILOSOPHY.md](../DESIGN_PHILOSOPHY.md).

---

## Table of Contents

1. [Responsibility](#responsibility)
2. [Public Interface](#public-interface)
3. [Data Types](#data-types)
4. [Built-in Sources](#built-in-sources)
5. [Concurrency Model](#concurrency-model)
6. [Retry and Error Handling](#retry-and-error-handling)
7. [Configuration Reference](#configuration-reference)
8. [Adding a New Source](#adding-a-new-source)
9. [Testing Strategy](#testing-strategy)

---

## Responsibility

The Extractor has exactly one job: **emit `RawFrame` objects into Queue A as fast and as reliably as the underlying data source allows.**

It is explicitly **not** responsible for:

- Cleaning, normalizing, or transforming images — that belongs to the Preprocessor
- Deciding which images are scientifically interesting — that belongs to the user's query
- Caching or deduplicating frames — that is the user's concern; the Extractor is stateless
- Translating between image formats once loaded — `RawFrame` carries the raw array as-is

This single-responsibility discipline is what allows the Extractor to be replaced wholesale (e.g., to support a new telescope) without touching any other part of the pipeline.

---

## Public Interface

The Extractor  exposes one abstract base class and two concrete reference implementations. Every data source — built-in or contributed — implements this interface.

```python
from abc import ABC, abstractmethod
from typing import AsyncIterator
from flux.extractor.types import RawFrame, SourceQuery

class DataSource(ABC):
    """Abstract interface for any data source feeding Phase A."""

    @abstractmethod
    async def fetch(self, query: SourceQuery) -> AsyncIterator[RawFrame]:
        """Yield RawFrame objects matching the given query.

        Implementations MUST:
          - Be async generators (use `async def` + `yield`).
          - Yield each frame independently — no batching at this layer.
          - Attach complete metadata to every RawFrame.
          - Raise SourceUnavailableError on unrecoverable errors.
          - Allow cancellation at any await point.
        """
        ...

    @abstractmethod
    def estimate_size(self, query: SourceQuery) -> int | None:
        """Return the expected number of frames for this query, or None
        if unknown. Used for progress bars and queue sizing heuristics."""
        ...

    async def close(self) -> None:
        """Release any held resources (file handles, network sessions).
        Default implementation is a no-op. Override if needed."""
        return None
```

Two design notes about this interface:

The `fetch` method returns an `AsyncIterator`, not a list. This is non-negotiable — it is how FLUX achieves bounded-memory streaming for arbitrarily large surveys. Implementations that load everything into a list before yielding violate the streaming contract documented in [DESIGN_PHILOSOPHY.md](../DESIGN_PHILOSOPHY.md).

The `estimate_size` method is allowed to return `None` for sources that cannot know their size in advance (live telescope feeds, unbounded queries). Phase B and Phase C must tolerate `None` — they do, by falling back to indefinite progress reporting.

---

## Data Types

Three types form the public data model of Phase A.

### `RawFrame`

The unit of data that crosses the boundary from Phase A to Phase B. Defined as a frozen Pydantic model:

```python
from pydantic import BaseModel, ConfigDict
import numpy as np

class RawFrame(BaseModel):
    model_config = ConfigDict(arbitrary_types_allowed=True, frozen=True)

    frame_id: str                  # globally unique identifier
    image: np.ndarray              # raw pixel array, shape (C, H, W) or (H, W)
    metadata: FrameMetadata        # see below
    provenance: Provenance         # see below
```

The `frozen=True` constraint is intentional. Once a `RawFrame` leaves the Extractor, it is **immutable**. The Preprocessor may produce a new `ProcessedFrame` derived from it, but the `RawFrame` itself never mutates. This eliminates an entire category of bugs related to shared mutable state across async tasks.

### `FrameMetadata`

```python
class FrameMetadata(BaseModel):
    object_id: str | None          # source catalog identifier, if any
    filter_band: str | None        # e.g. "g", "r", "i", "z", "y"
    exposure_time: float | None    # seconds
    pixel_scale: float | None      # arcseconds per pixel
    ra: float | None               # right ascension, degrees
    dec: float | None              # declination, degrees
    extra: dict[str, Any] = {}     # source-specific fields
```

The standard fields are nullable because not every data source provides them. The `extra` dict is the escape hatch for source-specific information that does not fit the standard schema (e.g., simulation parameters for synthetic data, observation conditions for real data).

### `Provenance`

```python
class Provenance(BaseModel):
    source_name: str               # e.g. "MockButlerSource"
    source_version: str            # version of the source plugin
    fetched_at: datetime           # UTC timestamp
    location: str                  # URI, file path, or query string
    retry_count: int = 0           # how many retries it took to fetch
```

Provenance is what makes the error bus actionable. When a frame fails downstream, the user can trace back exactly where it came from and how. This is non-optional in scientific software — reproducibility requires provenance.

### `SourceQuery`

The input to the `fetch` method. Different sources accept different query types, but all subclass:

```python
class SourceQuery(BaseModel):
    """Base class for source-specific queries."""
    limit: int | None = None       # max frames to fetch, None = no limit
```

Concrete sources extend this (e.g., `ButlerQuery` adds `filter_bands`, `tract`, `patch`).

---

## Built-in Sources

FLUX v0.1 ships with two reference `DataSource` implementations.

### `MockButlerSource`

A standalone implementation that mimics the LSST Butler API over a local directory of `.npy` files. Its purpose is to allow the entire pipeline to be developed, tested, and demonstrated without access to the real Rubin Observatory data — and without needing to install the heavy `lsst.daf.butler` package.

The query semantics replicate a useful subset of Butler:

```python
query = ButlerQuery(
    collection="DP0_mock",
    filter_bands=["g", "r", "i"],
    tract=4225,
    patch="3,4",
    limit=1000,
)
```

Internally, `MockButlerSource` reads a manifest file (`manifest.yaml`) that maps Butler-style coordinates to local file paths. This manifest is generated once by a helper script (`flux.tools.build_mock_manifest`) and committed alongside the test datasets.

When real Butler support is added in v0.2 (as `LSSTButlerSource`), it will share the same `ButlerQuery` schema, so user code does not change — only the source class registered in the YAML configuration.

### `GenericDirectorySource`

A simpler implementation for any flat directory of supported files. Useful for adapting public datasets from HuggingFace or Google Drive (such as the ML4SCI DeepLense classification dataset) without a Butler-shaped wrapper.

```python
query = DirectoryQuery(
    root_path="./data/deeplense_classification",
    extensions=[".npy"],
    label_from_subfolder=True,    # use parent dir name as label
    limit=None,
)
```

Supported file types in v0.1: `.npy`, `.fits`. Future versions will add `.png` and `.jpg` for community-contributed datasets.

---

## Concurrency Model

The Extractor uses `asyncio` for I/O concurrency. Multiple frames are fetched in parallel up to a configurable limit, controlled by `max_concurrent_fetches` in the source configuration.

The standard pattern in every `DataSource.fetch` implementation is:

```python
async def fetch(self, query):
    semaphore = asyncio.Semaphore(self.config.max_concurrent_fetches)

    async def _fetch_one(location):
        async with semaphore:
            return await self._load_frame(location)

    locations = self._resolve_locations(query)
    tasks = [_fetch_one(loc) for loc in locations]

    for coro in asyncio.as_completed(tasks):
        frame = await coro
        if frame is not None:
            yield frame
```

The semaphore caps concurrent in-flight fetches so memory usage stays bounded. `as_completed` yields frames in completion order, not request order — this is intentional: it maximizes throughput and the downstream pipeline does not depend on order.

If your source benefits from ordered output (e.g., time-series processing), set `max_concurrent_fetches: 1` in the configuration. The trade-off is throughput.

---

## Retry and Error Handling

Failures in the Extractor are categorized as:

- **Transient errors** — network timeouts, file system stalls, temporary 5xx responses. Retried with exponential backoff. Configurable: `max_retries` (default 3), `initial_backoff_seconds` (default 1.0), `backoff_multiplier` (default 2.0).
- **Data integrity errors** — corrupted files, unreadable headers, NaN-only arrays. Frame is dropped, error logged, pipeline continues. Not retried — retrying a corrupted file does not uncorrupt it.
- **Source unavailable errors** — Butler database is down, root directory does not exist, credentials are invalid. Pipeline fails fast at the configuration validation step, before any data is fetched. Not retried.

Every dropped frame is emitted to the structured error bus with full provenance:

```json
{
  "phase": "extractor",
  "source": "MockButlerSource",
  "frame_id": "DP0_g_band_00042",
  "error_type": "CorruptedFileError",
  "message": "numpy.load failed: invalid magic number",
  "retry_count": 0,
  "dropped": true,
  "location": "./data/mock_butler/g/DP0_g_band_00042.npy",
  "timestamp": "2025-04-12T18:33:21Z"
}
```

The error bus implementation is shared across all three phases and lives in `flux/errors.py`.

---

## Configuration Reference

Every Extractor source is configured via a `source` section in the pipeline YAML:

```yaml
source:
  type: MockButlerSource          # name registered in the source registry
  collection: DP0_mock
  filter_bands: [g, r, i]
  tract: 4225
  patch: "3,4"
  limit: 1000
  max_concurrent_fetches: 8
  max_retries: 3
  initial_backoff_seconds: 1.0
  backoff_multiplier: 2.0
  manifest_path: ./data/mock_butler/manifest.yaml
```

The `type` field selects the implementation class; the remaining fields are parsed by the corresponding Pydantic config class registered with that type. Unknown fields raise a validation error at startup, never silently ignored.

The full schema reference is auto-generated from the Pydantic models and lives in [CONFIGURATION.md](../CONFIGURATION.md).

---

## Adding a New Source

Adding support for a new data source is the most common extension scenario for FLUX. The full procedure:

**Step 1.** Create a new module under `flux/extractor/` (e.g., `flux/extractor/my_telescope.py`).

**Step 2.** Define a query schema:

```python
from flux.extractor.types import SourceQuery

class MyTelescopeQuery(SourceQuery):
    survey_id: str
    night: date
```

**Step 3.** Define a configuration schema:

```python
from flux.config.base import SourceConfig

class MyTelescopeConfig(SourceConfig):
    type: Literal["MyTelescopeSource"] = "MyTelescopeSource"
    api_endpoint: str
    api_key_env: str = "MY_TELESCOPE_API_KEY"
```

**Step 4.** Implement the source:

```python
from flux.extractor.base import DataSource

class MyTelescopeSource(DataSource):
    def __init__(self, config: MyTelescopeConfig):
        self.config = config

    async def fetch(self, query: MyTelescopeQuery):
        # Your async fetch logic here
        ...

    def estimate_size(self, query):
        return None  # if unknown
```

**Step 5.** Register the source:

```python
# In flux/extractor/registry.py
from .my_telescope import MyTelescopeSource, MyTelescopeConfig

SOURCE_REGISTRY.register(
    name="MyTelescopeSource",
    source_class=MyTelescopeSource,
    config_class=MyTelescopeConfig,
)
```

**Step 6.** Write tests against the contract:

The base test class `tests/extractor/test_source_contract.py` runs every concrete source against a standard suite of behavior tests (immutability, cancellation, error propagation, provenance completeness). Your implementation passes by inheriting from `BaseSourceContractTest` and parametrizing with your config.

This is what allows new sources to be added with confidence — the contract tests guarantee the behavioral invariants the rest of the pipeline depends on.

---

## Testing Strategy

The Extractor is tested at three levels:

**Unit tests** verify the internal helpers of each source implementation in isolation. For `MockButlerSource`, this means manifest parsing, path resolution, file loading. These tests are fast and have no external dependencies.

**Contract tests** verify that every concrete `DataSource` satisfies the abstract contract. This includes: every yielded frame is a valid `RawFrame`, cancellation does not leak resources, retry logic respects backoff timing, provenance is always populated, the source correctly handles empty query results.

**Integration tests** verify that the Extractor cooperates correctly with the rest of the pipeline. These tests spin up a `MockButlerSource` against a small fixture dataset, drive it through Queue A into a stub Preprocessor, and assert frame count and metadata propagation.

The test layout mirrors the source layout:

```
tests/
└── extractor/
    ├── test_source_contract.py       # contract tests (parametrized)
    ├── test_mock_butler.py           # unit tests for MockButlerSource
    ├── test_generic_directory.py     # unit tests for GenericDirectorySource
    └── test_extractor_integration.py # integration with the pipeline
```

Coverage target for this module: **>90%**, given that it is the boundary between FLUX and the unpredictable outside world.

---

## See also

- [ARCHITECTURE.md](../ARCHITECTURE.md) — system-level view of the three phases
- [DESIGN_PHILOSOPHY.md](../DESIGN_PHILOSOPHY.md) — principles guiding source design
- [modules/preprocessor.md](preprocessor.md) — what consumes the output of the Extractor
- [CONFIGURATION.md](../CONFIGURATION.md) — full YAML schema reference
- [CONTRIBUTING.md](../../CONTRIBUTING.md) — code style and testing conventions