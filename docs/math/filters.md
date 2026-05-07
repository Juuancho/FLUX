# Mathematical Reference: Filters

This document provides the mathematical formulation of the noise-handling filters used in FLUX's preprocessing pipeline. Four filters are documented here. The fifth, **Richardson-Lucy PSF deconvolution**, has its own document due to its complexity: see [psf_deconvolution.md](psf_deconvolution.md).

For the API contract and configuration of each filter, see [modules/preprocessor.md](../modules/preprocessor.md). For the enforced ordering between filters, see the [Ordering Rules](../modules/preprocessor.md#ordering-rules) section.

---

## Table of Contents

1. [Notation](#notation)
2. [Gaussian Filter](#gaussian-filter)
3. [Median Filter](#median-filter)
4. [Sigma Clipping](#sigma-clipping)
5. [Wavelet Denoising](#wavelet-denoising)
6. [References](#references)

---

## Notation

Throughout this document:

- $I(x, y)$ denotes a 2D input image with pixel coordinates $(x, y)$
- $I'(x, y)$ denotes the corresponding output of a transformation
- $K(u, v)$ denotes a convolution kernel of size $(2k+1) \times (2k+1)$
- $\mu$ and $\sigma$ denote the mean and standard deviation of pixel intensities
- $*$ denotes 2D convolution
- For multi-channel data $I(x, y, c)$ with channel index $c$, all filters are applied independently per channel unless stated otherwise

---

## Gaussian Filter

### Definition

The Gaussian filter convolves the input image with a discretized 2D Gaussian kernel:

$$
G(u, v; \sigma) = \frac{1}{2\pi\sigma^2} \exp\left(-\frac{u^2 + v^2}{2\sigma^2}\right)
$$

The filtered image is:

$$
I'(x, y) = (I * G)(x, y) = \sum_{u=-k}^{k} \sum_{v=-k}^{k} I(x - u, y - v) \cdot G(u, v; \sigma)
$$

In practice, the continuous Gaussian is sampled on the integer kernel grid and then normalized so that the kernel sums to 1, ensuring the filter preserves the total flux of the image.

### Kernel Size

The kernel half-width $k$ should be at least $3\sigma$ to avoid truncation artifacts. FLUX's default behavior, when `kernel_size: 0` is configured, is to compute:

$$
k = \lceil 3\sigma \rceil, \quad \text{kernel\_size} = 2k + 1
$$

For example, $\sigma = 1.0$ produces a $7 \times 7$ kernel; $\sigma = 1.5$ produces a $9 \times 9$ kernel.

### Boundary Handling

Convolution at image borders requires defining values outside the image. FLUX supports three modes:

- `reflect` (default): mirror pixels across the boundary, $I(-x, y) = I(x, y)$
- `constant`: pad with a fixed value (zero by default)
- `nearest`: extend edge pixels, $I(-x, y) = I(0, y)$

`reflect` is preferred for astronomical images because it does not introduce artificial sharp edges where the source meets the boundary.

### Properties

The Gaussian filter is **linear**, **separable**, and **isotropic**:

- *Linear*: $(\alpha I_1 + \beta I_2) * G = \alpha (I_1 * G) + \beta (I_2 * G)$
- *Separable*: 2D convolution can be implemented as two successive 1D convolutions, reducing complexity from $O(k^2)$ to $O(k)$ per pixel
- *Isotropic*: rotationally symmetric, treats all directions equally

These properties are exploited in the FLUX implementation for performance.

### Use Case

Effective for general-purpose smoothing of additive Gaussian noise — the dominant noise model for read noise in CCD sensors at moderate exposure times. **Not** appropriate for impulse noise (dead pixels, cosmic rays), which should be handled by the median filter or sigma clipping.

---

## Median Filter

### Definition

The median filter is a non-linear filter that replaces each pixel with the median of its neighborhood:

$$
I'(x, y) = \mathrm{median} \left\{ I(x + u, y + v) \;\middle|\; -k \leq u, v \leq k \right\}
$$

where the median is computed over a $(2k+1) \times (2k+1)$ window. For a window of $n = (2k+1)^2$ pixels, the median is the value at position $\lceil n/2 \rceil$ in the sorted list of the window's pixel values.

### Why Median Beats Gaussian for Impulse Noise

Consider a pixel corrupted by a cosmic ray: its value may be 100x the local mean. Under Gaussian smoothing, that outlier is *averaged* with its neighbors, contaminating the entire neighborhood by a fraction of its energy. Under median filtering, the outlier is *discarded*, because the median is dominated by the bulk of normal-valued pixels in the window.

Formally, the median is the **maximum-likelihood estimator of the central tendency under Laplace-distributed noise**, which has heavier tails than Gaussian noise. Impulse noise is well-modeled by such heavy-tailed distributions.

### Edge Preservation

A key property of the median filter is that it preserves sharp edges, unlike Gaussian smoothing, which blurs them. If a window straddles a sharp transition, the median selects a value from one side of the transition, not an interpolated mean. Mathematically, the median of the window is a member of the window — never a synthesized value — so it cannot create intermediate intensities that do not exist in the original image.

This property is critical for gravitational lens detection: the thin arcs that signal a strong lens are sharp, low-contrast features that Gaussian filtering can wash out.

### Computational Cost

Naive implementation requires sorting $(2k+1)^2$ values per pixel, giving $O(k^2 \log k)$ per pixel. FLUX uses the histogram-based sliding-window algorithm of Huang, Yang, and Tang (1979), which achieves $O(k)$ per pixel for integer-valued inputs and $O(k \log k)$ for floating-point inputs.

### Limitations

The median filter does not handle dense noise well — when more than 50% of a window is corrupted, the median itself becomes a corrupted value. For severe contamination, sigma clipping with `iterations > 1` is more effective.

---

## Sigma Clipping

### Definition

Sigma clipping is an iterative outlier rejection method. At each iteration, pixels whose values deviate from the local mean by more than $\sigma_t$ standard deviations are flagged and replaced. The process is repeated until no new outliers are found or a maximum iteration count is reached.

Let $\Omega$ denote the set of all pixel coordinates, and $M_i \subseteq \Omega$ the set of unmasked pixels at iteration $i$. The procedure is:

**Initialize:** $M_0 = \Omega$ (all pixels unmasked).

**Iterate** for $i = 0, 1, 2, \ldots$:

1. Compute statistics over the unmasked set:
   $$
   \mu_i = \frac{1}{|M_i|} \sum_{(x,y) \in M_i} I(x, y), \qquad
   \sigma_i = \sqrt{ \frac{1}{|M_i|} \sum_{(x,y) \in M_i} \left( I(x, y) - \mu_i \right)^2 }
   $$

2. Identify outliers:
   $$
   M_{i+1} = \left\{ (x, y) \in M_i \;\middle|\; \left| I(x, y) - \mu_i \right| \leq \sigma_t \cdot \sigma_i \right\}
   $$

3. **Stop** if $M_{i+1} = M_i$ (converged) or $i + 1 \geq i_{\max}$.

**Replace** each pixel in $\Omega \setminus M_{\mathrm{final}}$ (the rejected pixels) according to the configured `replace_with` strategy:

- `median`: substitute the median of the local $5 \times 5$ neighborhood
- `mean`: substitute $\mu_{\mathrm{final}}$
- `zero`: substitute 0

### Choice of $\sigma_t$

The threshold $\sigma_t$ controls aggressiveness. For a perfectly Gaussian noise distribution, the fraction of pixels rejected as a function of $\sigma_t$ follows the Q-function:

$$
P\left( \left| X \right| > \sigma_t \right) = 2 \cdot Q(\sigma_t) = \mathrm{erfc}\left( \sigma_t / \sqrt{2} \right)
$$

| $\sigma_t$ | Expected rejection rate |
|---|---|
| 1.0 | 31.7% |
| 2.0 | 4.55% |
| 2.5 | 1.24% |
| 3.0 | 0.27% |
| 3.5 | 0.047% |

The FLUX default of $\sigma_t = 3.0$ corresponds to rejecting roughly 1 in 370 pixels under pure Gaussian noise. Real astronomical images contain features whose pixels are legitimately bright (the lensed source itself, foreground galaxies), so the actual rejection rate on real data is typically 0.5–2%.

### Why Iterate

A single pass of clipping is biased: extreme outliers inflate $\sigma_0$, which causes the threshold $\sigma_t \cdot \sigma_0$ to be too generous, and many real outliers slip through. Iteration converges $\sigma_i$ toward the true noise standard deviation as outliers are removed, sharpening the threshold.

In practice, 2–3 iterations are sufficient for typical CCD data. The FLUX default is 3.

### Use Case

The standard tool for cosmic ray rejection in single-exposure images. Used as the first filter in the FLUX preprocessing chain because outliers contaminate the statistics that all subsequent filters depend on.

---

## Wavelet Denoising

### Definition

Wavelet denoising decomposes the image into multiple resolution levels using a wavelet transform, applies a thresholding operation to the detail coefficients to suppress noise, and reconstructs the image. The full procedure is:

**Decomposition.** The 2D discrete wavelet transform decomposes $I$ into one approximation image $A_L$ (the low-frequency content at the deepest level $L$) and a set of detail images $\{D_\ell^h, D_\ell^v, D_\ell^d\}$ for each level $\ell = 1, 2, \ldots, L$, capturing horizontal, vertical, and diagonal high-frequency structure respectively:

$$
I \;\;\xrightarrow{\;\;\mathrm{DWT}\;\;}\;\; \left( A_L,\; \{D_\ell^h, D_\ell^v, D_\ell^d\}_{\ell=1}^{L} \right)
$$

**Thresholding.** Each detail coefficient $d$ is replaced by $T(d, \lambda)$, where $\lambda$ is a level-dependent threshold and $T$ is the thresholding function:

- **Soft thresholding:**
  $$
  T_{\mathrm{soft}}(d, \lambda) = \mathrm{sign}(d) \cdot \max\left( |d| - \lambda,\; 0 \right)
  $$

- **Hard thresholding:**
  $$
  T_{\mathrm{hard}}(d, \lambda) = \begin{cases} d & \text{if } |d| > \lambda \\ 0 & \text{otherwise} \end{cases}
  $$

**Reconstruction.** The thresholded coefficients are passed through the inverse DWT to produce the denoised image $I'$.

### Choice of Wavelet Basis

The wavelet basis determines the shape of the analyzing function. FLUX supports the following families:

- `db4`, `db8`: Daubechies wavelets with 4 and 8 vanishing moments. Compact support, asymmetric. Widely used for general-purpose denoising.
- `sym4`: Symlets, near-symmetric variant of Daubechies. Reduces phase distortion.
- `coif1`: Coiflets, with vanishing moments for both the wavelet and scaling function. Higher computational cost.

For gravitational lens images, `db4` is recommended as a balance between locality (preserving thin arcs) and smoothness (suppressing noise). `coif1` may be preferable when fine astrometric accuracy is important.

### Threshold Selection

FLUX supports three threshold-selection methods:

**Universal (VisuShrink).** Donoho and Johnstone's universal threshold:
$$
\lambda_{\mathrm{visu}} = \sigma_n \sqrt{2 \ln N}
$$
where $\sigma_n$ is the estimated noise standard deviation and $N$ is the number of pixels. Conservative — tends to over-smooth.

**Bayesian (BayesShrink).** Per-subband threshold derived from a generalized Gaussian model of the wavelet coefficients:
$$
\lambda_{\mathrm{bayes}} = \frac{\sigma_n^2}{\sigma_X}
$$
where $\sigma_X$ is the estimated standard deviation of the noiseless signal in that subband. Adaptive — preserves more detail than VisuShrink. **FLUX default.**

**Fixed.** A user-specified value, useful when noise statistics are known a priori from instrument calibration.

The noise standard deviation $\sigma_n$ is estimated robustly from the diagonal detail coefficients at the finest level using the median absolute deviation:

$$
\sigma_n = \frac{\mathrm{median}\left( \left| D_1^d \right| \right)}{0.6745}
$$

The factor 0.6745 is the constant that makes this estimator unbiased for Gaussian-distributed coefficients.

### Why Wavelet over Fourier

Fourier-based denoising assumes that noise and signal occupy distinct frequency bands. For natural images this is rarely true: edges and small features have significant high-frequency content that overlaps with noise. Wavelet denoising localizes both in space and in frequency simultaneously, allowing high-frequency signal at one location to be preserved while high-frequency noise at another location is suppressed.

For gravitational lens images, the thin arc of a strong lens is a high-frequency feature spatially confined to a small region of the image. Fourier filtering would either preserve it along with noise or destroy it; wavelets can preserve the arc while denoising the surrounding sky.

### Use Case

Final stage of the FLUX preprocessing chain when high-quality denoising is needed and the previous stages have already removed outliers and gross noise. Wavelet denoising is computationally more expensive than Gaussian or median filtering, so it is typically reserved for cases where preserving fine structure is critical.

---

## References

Books, papers, and software relied upon in the formulations above.

**Gaussian and Median filters.**
Gonzalez, R. C. & Woods, R. E. *Digital Image Processing*, 4th edition. Pearson, 2018. Chapters 3 and 5.

**Median filter sliding-window algorithm.**
Huang, T., Yang, G., & Tang, G. *A Fast Two-Dimensional Median Filtering Algorithm.* IEEE Transactions on Acoustics, Speech, and Signal Processing, 1979.

**Sigma clipping in astronomy.**
Stetson, P. B. *DAOPHOT: A Computer Program for Crowded-Field Stellar Photometry.* Publications of the Astronomical Society of the Pacific, 1987. Foundational reference for iterative clipping in astronomical photometry.

**Wavelet denoising — VisuShrink and BayesShrink.**
Donoho, D. L. & Johnstone, I. M. *Ideal Spatial Adaptation by Wavelet Shrinkage.* Biometrika, 1994.
Chang, S. G., Yu, B., & Vetterli, M. *Adaptive Wavelet Thresholding for Image Denoising and Compression.* IEEE Transactions on Image Processing, 2000.

**Daubechies wavelets.**
Daubechies, I. *Ten Lectures on Wavelets.* SIAM, 1992.

**Implementation references.**
The FLUX implementation builds on `scipy.ndimage` (Gaussian, median), `astropy.stats.sigma_clip`, and `pywavelets` (DWT and thresholding).

---

## See also

- [psf_deconvolution.md](psf_deconvolution.md) — the fifth filter, with its own dedicated document
- [normalization.md](normalization.md) — Z-score, min-max, and per-channel normalization
- [modules/preprocessor.md](../modules/preprocessor.md) — how these filters are composed in the pipeline