# Contributing to FLUX

Contributions to FLUX are welcome. This document describes the development workflow, code style, testing requirements, and the review process.

For significant architectural changes, see [RFC_PROCESS.md](docs/RFC_PROCESS.md). For the project's design principles that contributions must respect, see [DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md).

---

## Table of Contents

1. [Ways to Contribute](#ways-to-contribute)
2. [Development Setup](#development-setup)
3. [Branching and Pull Requests](#branching-and-pull-requests)
4. [Code Style](#code-style)
5. [Type Checking](#type-checking)
6. [Testing Requirements](#testing-requirements)
7. [Documentation Requirements](#documentation-requirements)
8. [Commit Message Conventions](#commit-message-conventions)
9. [Adding a New Plugin](#adding-a-new-plugin)
10. [Review Process](#review-process)

---

## Ways to Contribute

Contributions fall into four categories. Each has different expectations.

**Bug reports.** File an issue describing the bug, the FLUX version, the configuration that reproduces it, and the observed vs expected behavior. Configurations should be minimized: a configuration with one filter that reproduces the bug is more useful than one with all five.

**Documentation improvements.** Typo fixes, clarifications, missing examples, broken cross-references. These do not require an RFC and are usually merged quickly. Documentation-only PRs may skip the test suite if no code changed.

**New plugins.** New `DataSource`, `Transform`, or `TaskAdapter` implementations. The procedure is described in [Adding a New Plugin](#adding-a-new-plugin) below. New plugins must include contract tests, mathematical or empirical justification, and updates to the relevant module documentation.

**Architectural changes.** Modifications to the three-phase pipeline structure, the queue model, the configuration schema, or the public API. These require an RFC before implementation. See [RFC_PROCESS.md](docs/RFC_PROCESS.md).

---

## Development Setup

Clone the repository and install in editable mode with development dependencies:

```bash
git clone https://github.com/Juuancho/FLUX.git
cd flux

python -m venv .venv
source .venv/bin/activate          # Linux / macOS
.venv\Scripts\activate             # Windows

pip install -e ".[dev]"
```

The `[dev]` extra installs:

- `pytest` and `pytest-cov` for testing
- `pytest-asyncio` for async test support
- `black` for code formatting
- `ruff` for linting
- `mypy` for type checking
- `mkdocs` and `mkdocs-material` for local documentation builds

Verify the setup:

```bash
pytest                              # all tests should pass
black --check flux tests            # formatting should be clean
ruff check flux tests               # no linting errors
mypy flux                           # no type errors
```

If any of these commands fail on a fresh clone of `main`, that is itself a bug worth reporting.

---

## Branching and Pull Requests

The default branch is `main`. All work happens on feature branches.

**Branch naming.** Use the pattern `<type>/<short-description>`, where `<type>` matches the commit prefix (see [Commit Message Conventions](#commit-message-conventions)):

- `feat/wavelet-denoiser`
- `fix/preprocessor-empty-batch`
- `docs/clarify-cutout-config`
- `test/extractor-cancellation`

**Branch from `main`.** Always start from the latest `main`:

```bash
git checkout main
git pull
git checkout -b feat/my-new-filter
```

**Keep branches focused.** A pull request should address one concern. PRs that mix unrelated changes (a bug fix and a new feature, for example) are returned with a request to split.

**Open PRs as early as possible.** Draft pull requests are encouraged for early feedback. Mark the PR as draft, push commits as work progresses, and convert to ready-for-review when complete.

**Rebase, don't merge.** When `main` advances, rebase the feature branch:

```bash
git fetch origin
git rebase origin/main
```

Force-pushing to a feature branch is acceptable and expected. The merge into `main` uses squash-and-merge to keep the project history linear.

---

## Code Style

**Formatting.** All code is formatted with `black` using the default settings (line length 88). The formatter is run as a pre-commit hook and as a CI check; PRs with unformatted code are not merged.

```bash
black flux tests
```

**Linting.** `ruff` enforces a curated set of rules including unused imports, undefined names, mutable default arguments, and Python anti-patterns. The configuration is in `pyproject.toml` under `[tool.ruff]`.

```bash
ruff check flux tests
ruff check flux tests --fix         # auto-fix where possible
```

**Imports.** Imports are sorted in three groups separated by blank lines: standard library, third-party, local. Within each group, imports are alphabetical. `ruff` enforces this via the `I` rule set.

**Naming conventions.**

- Classes: `PascalCase` (`DataSource`, `MockButlerSource`, `RawFrame`)
- Functions and variables: `snake_case` (`fetch_frame`, `max_concurrent_fetches`)
- Constants: `UPPER_SNAKE_CASE` (`DEFAULT_BATCH_SIZE`)
- Private members: leading underscore (`_internal_helper`)
- Type variables: short `PascalCase` (`T`, `FrameType`)

**Docstrings.** All public classes, methods, and module-level functions have docstrings in Google style. Internal helpers (leading underscore) may omit docstrings if their purpose is obvious from the name and signature.

```python
def assemble_batch(self, frames: list[ProcessedFrame]) -> tuple[Tensor, ...]:
    """Combine a list of ProcessedFrames into a model-ready batch.

    Args:
        frames: List of preprocessed frames to batch together.

    Returns:
        Tuple of tensors matching the adapter's output_signature.

    Raises:
        BatchAssemblyError: If frames have inconsistent shapes or dtypes.
    """
```

**Line breaks in long signatures.** When a function signature does not fit on one line, break after the opening parenthesis and put each argument on its own line:

```python
def __init__(
    self,
    queue: AsyncQueue,
    adapter: TaskAdapter,
    batch_size: int,
    prefetch_factor: int = 4,
):
    ...
```

---

## Type Checking

FLUX is a fully type-annotated codebase. Every public function, method, and class attribute carries type hints. Internal helpers should also be typed unless the types are trivial.

`mypy` is run in strict mode on the `flux` package:

```bash
mypy flux
```

PRs that introduce type errors are not merged. The test directory is checked under more permissive settings — type errors in tests are warnings, not failures.

For Pydantic models, prefer the V2 syntax with `model_config` and `Field` annotations rather than the legacy `Config` class:

```python
from pydantic import BaseModel, ConfigDict, Field

class GaussianFilterConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")

    sigma: float = Field(default=1.0, gt=0)
    kernel_size: int = Field(default=0, ge=0)
```

`extra="forbid"` is required on every config model — it is what causes unknown YAML fields to fail validation rather than be silently ignored.

---

## Testing Requirements

Every code change is accompanied by tests. The project follows three testing conventions.

**Unit tests** verify a single function or class in isolation, with all dependencies stubbed or mocked. Located alongside the module under `tests/`:

```
flux/preprocessor/filters/gaussian.py
tests/preprocessor/filters/test_gaussian.py
```

**Contract tests** verify that every concrete implementation of an abstract interface satisfies the interface's contract. Each interface (`DataSource`, `Transform`, `TaskAdapter`) has a base test class that is parametrized over all registered implementations.

When adding a new plugin, the contract test runs automatically once the plugin is registered. No additional test code is needed for the contract itself — only for plugin-specific behavior.

**Integration tests** verify cross-phase behavior, typically by running a small fixture dataset through a complete pipeline configuration.

### Coverage requirements

The project enforces a minimum overall coverage of **85%** via CI. Per-module targets are higher in critical paths:

- `flux/extractor/`: >90%
- `flux/preprocessor/`: >92%
- `flux/ml_bridge/`: >88%
- `flux/config/`: >95%

Coverage is measured by `pytest-cov`:

```bash
pytest --cov=flux --cov-report=html
open htmlcov/index.html             # macOS
xdg-open htmlcov/index.html         # Linux
```

PRs that drop coverage below the threshold are not merged. Tests for newly added code must be present in the same PR — coverage cannot be deferred to a future PR.

### Test guidelines

**One concept per test.** Each test verifies one behavior. Tests with multiple assertions about unrelated properties are split.

**Descriptive names.** Test names follow the pattern `test_<thing_under_test>_<scenario>_<expected_outcome>`:

```python
def test_gaussian_filter_with_zero_sigma_raises_validation_error(): ...
def test_extractor_continues_after_corrupted_frame(): ...
```

**Fixtures over setUp.** `pytest` fixtures replace traditional `setUp`/`tearDown` patterns. Shared fixtures live in `conftest.py` files at the appropriate scope.

**No network calls.** Tests do not access the network. Sources that would make network calls in production (a real Butler client, for example) are tested against local fixtures or mocks. CI runs in environments without internet access.

**Deterministic.** Tests that depend on randomness must seed the RNG. Tests that depend on time use `freezegun` or equivalent. Flaky tests are bugs and are reverted.

---

## Documentation Requirements

Every code change that affects user-visible behavior is accompanied by documentation updates in the same PR. The relevant documents:

- New configuration field → [CONFIGURATION.md](docs/CONFIGURATION.md)
- New filter or adapter → the relevant module spec under [docs/modules/](docs/modules/) and [docs/math/](docs/math/) for filters
- New public API → docstring on the symbol, plus updates to the relevant module spec
- New behavior visible from a tutorial perspective → [GETTING_STARTED.md](GETTING_STARTED.md)
- New runtime behavior with performance implications → [PERFORMANCE.md](docs/PERFORMANCE.md)

Documentation is verified by the CI pipeline:

- All cross-references between markdown files must resolve to existing files
- All code blocks tagged as `python` must be syntactically valid Python (parsed with `ast`, not executed)
- All YAML examples must validate against the corresponding Pydantic schema

PRs that introduce new features without corresponding documentation are returned with a request to add it.

---

## Commit Message Conventions

FLUX uses Conventional Commits. Every commit message follows the pattern:

```
<type>: <short description>

<optional longer body>

<optional footer>
```

The accepted types:

| Type | Used for |
|---|---|
| `feat` | New user-visible feature |
| `fix` | Bug fix |
| `docs` | Documentation-only change |
| `test` | Adding or modifying tests, no production code change |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `perf` | Code change that improves performance |
| `chore` | Build system, CI, dependency updates |
| `style` | Formatting changes only |

The short description is in lowercase, no trailing period, imperative mood ("add feature" not "added feature"). Hard limit: 72 characters.

The optional body explains *why* the change was made, not *what*. Reviewers can see *what* from the diff.

Example:

```
feat: add Richardson-Lucy deconvolution filter

The 2023 Model IV experiment showed that PSF blurring is a major
contributor to model performance loss on real telescope data.
Adding RL deconvolution to the preprocessing chain addresses this.

The implementation uses TV regularization by default to preserve the
sharp arcs characteristic of strong gravitational lenses.
```

PRs are squash-merged into `main`, so the squashed commit message becomes the project history. Maintainers may rewrite the squash message during merge to follow these conventions even if individual commits did not.

---

## Adding a New Plugin

The most common contribution path: adding a new `DataSource`, `Transform`, or `TaskAdapter`. Each follows the same five-step procedure.

### Step 1: Create the module

Create a new module under the appropriate directory:

- New source: `flux/extractor/<name>.py`
- New filter: `flux/preprocessor/filters/<name>.py`
- New adapter: `flux/ml_bridge/<name>.py`

### Step 2: Define the configuration schema

Subclass the appropriate config base and use `Literal` for the `type` discriminator:

```python
from typing import Literal
from flux.config.base import TransformConfig

class MyDenoiserConfig(TransformConfig):
    type: Literal["MyDenoiser"] = "MyDenoiser"
    strength: float = Field(default=0.5, gt=0)
```

### Step 3: Implement the plugin

Subclass the abstract base, implementing all required methods. Honor the contract — purity for transforms, async iteration for sources, deterministic batch assembly for adapters.

### Step 4: Register the plugin

Add the registration call in the relevant `registry.py`:

```python
# flux/preprocessor/registry.py
from .filters.my_denoiser import MyDenoiser, MyDenoiserConfig

TRANSFORM_REGISTRY.register(
    name="MyDenoiser",
    transform_class=MyDenoiser,
    config_class=MyDenoiserConfig,
    ordering_slot="wavelet",        # for transforms only
)
```

### Step 5: Tests and documentation

Three things must be present in the same PR:

1. **A test file** at the mirrored path (`tests/preprocessor/filters/test_my_denoiser.py`) inheriting from the appropriate contract test class plus implementation-specific tests.

2. **A configuration entry** in [CONFIGURATION.md](docs/CONFIGURATION.md) under the relevant section, with the field table.

3. **Mathematical or empirical justification.** For new filters, this is an entry in [docs/math/filters.md](docs/math/filters.md) (or a new file under `docs/math/` if the algorithm warrants its own document). For new sources or adapters, this is a paragraph in the corresponding module spec explaining what use case the plugin enables.

Plugins that lack any of these three are not merged.

---

## Review Process

A PR enters review when marked ready-for-review (no longer draft) and CI is green.

**Reviewer expectations.** Maintainers aim to provide initial feedback within 5 working days. Larger or more complex PRs may take longer. Contributors are encouraged to follow up if more than 10 working days pass without response.

**Review categories.** Reviewers may request changes for any of the following:

- Correctness — the change does not do what it claims to do
- Tests — coverage is insufficient or tests do not exercise the new behavior
- Documentation — user-visible changes lack corresponding doc updates
- Architecture — the change violates principles in [DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md)
- Style — formatting or naming inconsistent with the codebase

**Discussion.** Disagreements with review feedback are welcome. Open a comment thread, explain the reasoning, and the reviewer will respond. If consensus is not reached, the discussion is escalated to a second maintainer.

**Approval and merge.** A PR requires one approving review from a maintainer. After approval and green CI, a maintainer merges via squash-and-merge. The squashed commit message becomes part of the project history.

**Post-merge.** After merge, the contributor is added to the [contributors list](https://github.com/Juuancho/flux/graphs/contributors) automatically. Substantial contributions are also acknowledged in the changelog of the next release.

---

## See also

- [README.md](README.md) — project overview
- [DESIGN_PHILOSOPHY.md](docs/DESIGN_PHILOSOPHY.md) — principles every contribution must respect
- [RFC_PROCESS.md](docs/RFC_PROCESS.md) — process for significant architectural proposals
- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — system structure
- [ROADMAP.md](ROADMAP.md) — planned future work