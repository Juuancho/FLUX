# Mathematical Reference: Normalization

This document provides the mathematical formulation of the normalization strategies used in FLUX's preprocessing pipeline. Three strategies are documented here, along with the streaming statistics algorithm that makes them tractable on datasets larger than memory.

For the API contract and configuration of normalization, see [modules/preprocessor.md](../modules/preprocessor.md).

---

## Table of Contents

1. [Notation](#notation)
2. [Why Normalize](#why-normalize)
3. [Z-Score Normalization](#z-score-normalization)
4. [Min-Max Normalization](#min-max-normalization)
5. [Per-Channel Z-Score](#per-channel-z-score)
6. [Welford's Online Algorithm](#welfords-online-algorithm)
7. [References](#references)

---

## Notation

Throughout this document:

- $x_i$ denotes the value of the $i$-th pixel in the dataset, with $i = 1, 2, \ldots, N$
- $N$ denotes the total number of pixels considered
- $\mu$ denotes a mean, $\sigma$ denotes a standard deviation
- $x'$ denotes the normalized value of $x$
- For multi-channel data with $C$ channels, the channel index is denoted $c \in \{1, \ldots, C\}$

---

## Why Normalize

Neural networks trained on images expect inputs in a controlled numerical range. Untreated astronomical data violates this assumption in two ways: pixel values can span many orders of magnitude (from background sky at $\sim 10^{-3}$ to saturated stars at $\sim 10^5$), and different filter bands have systematically different brightness distributions.

Without normalization, the first layers of a network spend capacity learning to compensate for these scale differences instead of learning physically meaningful features. Empirically, omitting normalization leads to slow convergence, gradient instability, and poor generalization across datasets.

Normalization addresses three concrete problems:

1. **Unit-scale inputs.** Standardizing inputs to roughly unit variance keeps activations and gradients in numerically stable ranges.
2. **Zero-mean inputs.** Centering inputs at zero allows symmetric activation functions (ReLU, GELU) to operate efficiently on both sides of zero.
3. **Cross-band comparability.** When models consume multiple filter bands, per-band normalization ensures no band dominates simply because of its absolute brightness.

Different normalization strategies prioritize these goals differently. The choice depends on the downstream model and the structure of the data.

---

## Z-Score Normalization

### Definition

Z-score normalization, also known as standardization, transforms each pixel value into the number of standard deviations it lies from the mean:

$$
x'_i = \frac{x_i - \mu}{\sigma}
$$

where $\mu$ and $\sigma$ are the mean and standard deviation computed over a reference set of pixels:

$$
\mu = \frac{1}{N} \sum_{i=1}^{N} x_i, \qquad
\sigma = \sqrt{\frac{1}{N} \sum_{i=1}^{N} (x_i - \mu)^2}
$$

After Z-score normalization, the transformed pixels have mean 0 and standard deviation 1 (over the reference set).

### Reference Set Selection

A critical decision is *which* pixels to include in the reference set. FLUX supports three strategies, configured by `stats_source`:

- **`dataset`** — statistics are computed over the entire training dataset using Welford's online algorithm (see below). The same $\mu$ and $\sigma$ are then applied to every frame, including the test set. This is the standard choice for transfer learning.
- **`per_image`** — statistics are computed independently for each frame, so each frame is centered and scaled to unit variance individually. Useful when frames have wildly different brightness baselines and the model is expected to be invariant to absolute brightness.
- **`fixed`** — user-supplied $\mu$ and $\sigma$ values, useful when reproducing a published preprocessing recipe or when statistics from a calibration dataset should be reused.

Mixing strategies between training and inference is a common bug source. FLUX requires that the same `stats_source` be used at both stages and validates this at configuration time.

### When to Use

Z-score is appropriate when:

- The downstream model uses architectures designed for ImageNet-scale inputs (most CNNs and ViTs)
- Pixel values are approximately Gaussian after preprocessing
- Absolute pixel scale carries no physical meaning the model should preserve

It is **not** appropriate when:

- The model expects inputs strictly in $[0, 1]$ (some generative architectures, certain attention mechanisms)
- The data has a long-tailed distribution where a few extreme values dominate $\sigma$ (in which case sigma clipping should run upstream)

---

## Min-Max Normalization

### Definition

Min-max normalization scales each pixel into the unit interval $[0, 1]$:

$$
x'_i = \frac{x_i - x_{\min}}{x_{\max} - x_{\min}}
$$

where $x_{\min}$ and $x_{\max}$ are the minimum and maximum pixel values in the reference set.

To map into a different target range $[a, b]$:

$$
x'_i = a + (b - a) \cdot \frac{x_i - x_{\min}}{x_{\max} - x_{\min}}
$$

### Robust Variant

The naive min-max formulation is sensitive to outliers: a single saturated pixel can determine $x_{\max}$ and compress the rest of the data into a tiny fraction of the output range. FLUX defaults to the robust variant, where $x_{\min}$ and $x_{\max}$ are replaced by user-configurable percentiles (default: 1st and 99th):

$$
x'_i = \mathrm{clip}\left( \frac{x_i - p_{1}}{p_{99} - p_{1}}, \; 0, \; 1 \right)
$$

The `clip` operation forces values from the extreme tails to exactly 0 or 1 instead of producing values outside the target range.

### When to Use

Min-max is appropriate when:

- The downstream model expects bounded inputs (super-resolution networks, generative models)
- The dataset has already been outlier-cleaned by sigma clipping upstream
- Absolute pixel scale matters and Z-score would distort it

---

## Per-Channel Z-Score

### Definition

For multi-channel data with $C$ channels (e.g., $C = 3$ filter bands $g, r, i$), per-channel Z-score computes a separate mean and standard deviation for each channel:

$$
\mu_c = \frac{1}{N_c} \sum_{i \in \mathcal{C}_c} x_i, \qquad
\sigma_c = \sqrt{\frac{1}{N_c} \sum_{i \in \mathcal{C}_c} (x_i - \mu_c)^2}
$$

where $\mathcal{C}_c$ is the set of pixels in channel $c$. The transformed pixel is then:

$$
x'_i = \frac{x_i - \mu_{c(i)}}{\sigma_{c(i)}}
$$

where $c(i)$ is the channel of pixel $i$.

### Why Per-Channel for Astronomical Data

The three LSST filter bands $g$, $r$, $i$ correspond to different wavelength ranges and have systematically different brightness distributions:

- The $g$ band (blue, $\sim 470$ nm) captures bluer galaxies and is dimmer for typical lensing systems
- The $r$ band (red, $\sim 620$ nm) is the deepest LSST band and dominates in flux for most sources
- The $i$ band (near-infrared, $\sim 750$ nm) preserves redder structures

Applying a single global $\mu$ and $\sigma$ across all channels would underweight the $g$ band and overweight the $r$ band, suppressing the spectral information the model could otherwise exploit. Per-channel normalization preserves the relative within-channel structure while equalizing the cross-channel scale.

### Recommended Default

For multi-band astronomical data, per-channel Z-score is the recommended default. For single-channel data, it reduces to ordinary Z-score and produces identical results.

---

## Welford's Online Algorithm

### The Problem

Computing $\mu$ and $\sigma$ over a dataset of $N$ pixels naively requires either:

- Loading all $N$ pixels into memory, or
- Two passes over the data (one for $\mu$, one for $\sigma$)

Neither is acceptable for FLUX. The pipeline is designed to handle datasets that exceed available memory, and a two-pass scheme would force the data through Phase A twice. The streaming requirement is documented as a core principle in [DESIGN_PHILOSOPHY.md](../DESIGN_PHILOSOPHY.md#streaming-over-materialization).

The classical single-pass formula

$$
\sigma^2 = \frac{1}{N} \sum_{i=1}^{N} x_i^2 - \mu^2
$$

is mathematically correct but **numerically unstable** when $\sigma \ll \mu$, which is common in astronomical data: most pixels are background sky values close to a stable mean. Subtracting two large nearly-equal quantities causes catastrophic cancellation in floating-point arithmetic.

### The Algorithm

Welford (1962) gave a numerically stable single-pass recurrence. Define the running statistics after seeing $n$ samples:

- $M_n$: running mean
- $S_n$: running sum of squared deviations from the mean, $S_n = \sum_{i=1}^{n} (x_i - M_n)^2$

The recurrences are:

$$
M_n = M_{n-1} + \frac{x_n - M_{n-1}}{n}
$$

$$
S_n = S_{n-1} + (x_n - M_{n-1}) (x_n - M_n)
$$

with initial conditions $M_0 = 0$, $S_0 = 0$. The variance is then:

$$
\sigma^2 = \frac{S_n}{n}
$$

(or $\frac{S_n}{n-1}$ for the unbiased sample variance).

### Why It Works

Welford's recurrence avoids cancellation because it never computes $\sum x_i^2$ directly. Instead, it accumulates squared *deviations* from a continuously-updated mean. The intermediate quantities stay numerically small even when the underlying values are large.

A formal stability analysis shows that the error in $S_n$ grows as $O(\sqrt{n} \, \epsilon)$, where $\epsilon$ is machine epsilon. The naive formula's error grows as $O(n \, \epsilon)$ — a substantial difference at survey scale, where $n$ may exceed $10^{11}$ pixels.

### Implementation in FLUX

FLUX implements Welford's algorithm in `flux/preprocessor/normalize.py`. The streaming statistics object is maintained per-channel for `PerChannelZScore`:

```python
class WelfordAccumulator:
    def __init__(self):
        self.n = 0
        self.mean = 0.0
        self.m2 = 0.0  # this is S_n in the math above

    def update(self, x: np.ndarray):
        for value in x.flat:
            self.n += 1
            delta = value - self.mean
            self.mean += delta / self.n
            delta2 = value - self.mean
            self.m2 += delta * delta2

    @property
    def variance(self):
        return self.m2 / self.n if self.n > 0 else 0.0

    @property
    def std(self):
        return math.sqrt(self.variance)
```

The actual implementation is vectorized over array chunks rather than per-pixel, but the recurrence is identical.

### Parallel Welford

When statistics are computed in parallel across Dask workers, each worker maintains its own `WelfordAccumulator` and the partial results are merged at the end. The merge formula (Chan, Golub & LeVeque, 1979) for combining accumulators $A$ and $B$ into $A \cup B$:

$$
n_{AB} = n_A + n_B
$$

$$
\delta = M_B - M_A
$$

$$
M_{AB} = M_A + \delta \cdot \frac{n_B}{n_{AB}}
$$

$$
S_{AB} = S_A + S_B + \delta^2 \cdot \frac{n_A n_B}{n_{AB}}
$$

This merge is associative and commutative, so workers can finish in any order without affecting the final result.

---

## References

**Welford's algorithm.**
- Welford, B. P. *Note on a Method for Calculating Corrected Sums of Squares and Products.* Technometrics, 1962. The original publication.

**Parallel merge for streaming variance.**
- Chan, T. F., Golub, G. H., & LeVeque, R. J. *Updating Formulae and a Pairwise Algorithm for Computing Sample Variances.* Stanford Technical Report STAN-CS-79-773, 1979.

**Numerical stability analysis.**
- Higham, N. J. *Accuracy and Stability of Numerical Algorithms*, 2nd edition. SIAM, 2002. Chapter 1 covers the stability of variance computation in detail.

**Z-score in deep learning preprocessing.**
- LeCun, Y., Bottou, L., Orr, G. B., & Müller, K.-R. *Efficient BackProp.* In *Neural Networks: Tricks of the Trade*, Springer, 1998. Foundational reference for the role of input normalization in training deep networks.

**Normalization in modern astronomical ML pipelines.**
- Pearson, J. et al. *GraViT: Transfer Learning with Vision Transformers and MLP-Mixer for Strong Gravitational Lens Discovery.* MNRAS, 2025 (arXiv:2509.00226). Includes a study of how normalization choices interact with transfer learning performance on LSST-shaped data.
- Madireddy, S. et al. *A Modular Deep Learning Pipeline for Galaxy-Scale Strong Gravitational Lens Detection and Modeling.* arXiv:1911.03867 (LSST DESC). Discusses per-channel normalization in the context of LSST DC2 simulated multi-band data.

**Implementation references.**

The FLUX implementation builds on `numpy` for vectorized accumulation and `dask` for the parallel merge across distributed workers.

---

## See also

- [filters.md](filters.md) — the noise filters that should run before normalization
- [psf_deconvolution.md](psf_deconvolution.md) — Richardson-Lucy deconvolution
- [modules/preprocessor.md](../modules/preprocessor.md) — how normalization is composed in the pipeline
- [DESIGN_PHILOSOPHY.md](../DESIGN_PHILOSOPHY.md#streaming-over-materialization) — why streaming statistics matter