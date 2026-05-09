# Mathematical Reference: PSF Deconvolution

This document derives and specifies the Richardson-Lucy algorithm used in FLUX for Point Spread Function (PSF) deconvolution. It is the most mathematically involved transform in the preprocessing pipeline and warrants its own document.

For the API contract and configuration, see [modules/preprocessor.md](../modules/preprocessor.md). For other filters, see [filters.md](filters.md).

---

## Table of Contents

1. [Notation](#notation)
2. [The Imaging Equation](#the-imaging-equation)
3. [Why Naive Inversion Fails](#why-naive-inversion-fails)
4. [Bayesian Formulation](#bayesian-formulation)
5. [The Richardson-Lucy Iteration](#the-richardson-lucy-iteration)
6. [Convergence and Stopping](#convergence-and-stopping)
7. [Regularization](#regularization)
8. [PSF Calibration and Validity](#psf-calibration-and-validity)
9. [Numerical Implementation](#numerical-implementation)
10. [Limitations](#limitations)
11. [References](#references)

---

## Notation

- $f(x, y)$: the unknown true sky image (the quantity to recover)
- $g(x, y)$: the observed image (the quantity available to FLUX)
- $h(x, y)$: the Point Spread Function (PSF), assumed known and shift-invariant
- $n(x, y)$: a noise term
- $*$: 2D convolution
- $\hat{f}^{(k)}(x, y)$: the estimate of $f$ after $k$ iterations
- $h^*(x, y) = h(-x, -y)$: the spatially flipped PSF (the adjoint of convolution by $h$)

All images are assumed non-negative, since they represent photon counts.

---

## The Imaging Equation

A telescope does not record the sky directly. Light from a point source is spread by the atmosphere, the optical system, and the detector pixel response into an extended pattern called the **Point Spread Function**. Under the assumption that this spreading is shift-invariant across the field of view, the observed image is the convolution of the true image with the PSF, plus noise:

$$g(x, y) = (f * h)(x, y) + n(x, y)$$

The goal of deconvolution is to estimate $f$ given $g$ and $h$.

For optical telescopes, the PSF is dominated by atmospheric seeing and is typically modeled as a Moffat or Gaussian profile of width 0.6–1.5 arcseconds depending on observing conditions. For LSST DP0 simulations, the PSF is provided as a calibration image alongside the data.

---

## Why Naive Inversion Fails

The convolution theorem suggests an obvious inversion: in the Fourier domain, convolution becomes multiplication, so

$$\hat{f}(u, v) = \frac{G(u, v)}{H(u, v)}$$

where $G$, $H$, and $\hat{f}$ are the Fourier transforms of $g$, $h$, and the estimate of $f$.

This formulation has two fatal problems for astronomical data:

**Division by near-zero values.** The PSF acts as a low-pass filter — $H(u, v)$ becomes very small at high frequencies. Dividing by these tiny values amplifies any noise present at those frequencies by a huge factor. The result is dominated by amplified noise, not by the true image.

**No non-negativity constraint.** The true sky image is non-negative (it represents photon counts), but the Fourier-domain quotient produces negative values where noise dominates. These negative pixels are physically meaningless and harmful to downstream models.

Iterative methods like Richardson-Lucy address both problems by enforcing non-negativity at every step and by terminating before noise amplification dominates.

---

## Bayesian Formulation

Richardson-Lucy can be derived as the Maximum-Likelihood Estimator of $f$ under a Poisson noise model. This is the appropriate noise model for astronomical imaging, where each pixel counts photon arrivals.

### The likelihood

Under Poisson statistics, the probability of observing $g(x, y)$ photons at pixel $(x, y)$, given an expected rate of $(f * h)(x, y)$, is:

$$P(g(x, y) \mid f) = \frac{ [(f * h)(x, y)]^{g(x, y)} \exp(-(f * h)(x, y)) }{ g(x, y)! }$$

Assuming pixels are independent, the joint log-likelihood is:

$$\log L(f) = \sum_{x, y} \big[ g(x, y) \log (f * h)(x, y) - (f * h)(x, y) \big] + \text{const}$$

### Maximization

Maximizing $\log L(f)$ with respect to $f$ subject to $f \geq 0$ does not have a closed-form solution. The Expectation-Maximization (EM) procedure produces a multiplicative update rule that converges to the maximum-likelihood estimate. That update rule is the Richardson-Lucy iteration.

The full EM derivation is beyond the scope of this document; see Shepp & Vardi (1982) for the canonical proof in the context of medical imaging, where the same algorithm was independently derived.

---

## The Richardson-Lucy Iteration

### The update rule

Starting from an initial estimate $\hat{f}^{(0)}$ (typically a flat image equal to the mean of $g$), the iteration is:

$$\hat{f}^{(k+1)}(x, y) = \hat{f}^{(k)}(x, y) \cdot \left[ \frac{ g(x, y) }{ (\hat{f}^{(k)} * h)(x, y) } * h^*(x, y) \right]$$

where $h^*(x, y) = h(-x, -y)$ is the flipped PSF.

### Reading the formula

The iteration has a clean physical interpretation:

1. **Predict** the observation given the current estimate: compute $\hat{f}^{(k)} * h$.
2. **Compare** to the actual observation by taking the ratio $g / (\hat{f}^{(k)} * h)$. This ratio equals 1 where the prediction matches the data, $> 1$ where the model under-predicts, $< 1$ where it over-predicts.
3. **Backproject** the ratio through the flipped PSF (the adjoint operator) to map pixel-space corrections back to source-space corrections.
4. **Multiply** the current estimate by the backprojected correction.

The multiplicative form of the update guarantees that $\hat{f}^{(k)} \geq 0$ at every iteration, provided the initial estimate is non-negative. Non-negativity is preserved exactly, not just approximately.

### Properties

The iteration has three notable properties:

**Flux conservation.** If the PSF is normalized so that $\sum h(x, y) = 1$, then the total flux of the estimate is preserved across iterations: $\sum_{x, y} \hat{f}^{(k+1)} = \sum_{x, y} \hat{f}^{(k)}$. Photon counting is respected.

**Monotonic likelihood.** The log-likelihood is non-decreasing across iterations: $\log L(\hat{f}^{(k+1)}) \geq \log L(\hat{f}^{(k)})$. This is the EM monotonicity property and is one of the strongest theoretical guarantees in iterative imaging.

**No closed-form fixed point.** Unlike Wiener filtering, Richardson-Lucy has no closed-form solution. The number of iterations is itself a regularization parameter.

---

## Convergence and Stopping

Richardson-Lucy converges to the maximum-likelihood estimate of $f$ under Poisson noise, but **convergence is not the goal**. As iterations proceed, the algorithm fits the noise as well as the signal — a phenomenon called **noise amplification**. Run too long, the deconvolved image becomes a sharp but noisy mess.

The standard stopping strategies, in order of increasing sophistication:

**Fixed iteration count.** The simplest and most common: stop after a pre-determined number of iterations. The FLUX default is `iterations: 30`. This works because the optimal stopping point is empirically stable for a given PSF and noise level.

**Discrepancy principle.** Stop when the residual $\| g - \hat{f}^{(k)} * h \|$ falls to the level of the noise standard deviation. Useful when a reliable noise estimate is available; less reliable when noise is non-stationary.

**Cross-validation on held-out pixels.** Compute the likelihood on a randomly held-out fraction of pixels at each iteration and stop when it stops improving. The most rigorous strategy, but expensive to compute.

FLUX uses fixed iteration count by default and exposes the count as a configuration parameter. The recommended range is 20–50 iterations for typical optical data.

---

## Regularization

Beyond early stopping, FLUX supports two explicit regularization options that modify the update rule to penalize undesired solutions.

### Total Variation regularization

Total Variation (TV) regularization adds a penalty term proportional to the spatial gradient of the estimate:

$$R_{\text{TV}}(f) = \sum_{x, y} \sqrt{ (\partial_x f)^2 + (\partial_y f)^2 }$$

The penalty is small for piecewise-smooth images and large for noisy images, biasing the solution toward smooth reconstructions. The modified update rule (Dey et al., 2006) is:

$$\hat{f}^{(k+1)} = \frac{ \hat{f}^{(k)} }{ 1 - \lambda_{\text{TV}} \cdot \text{div}\left( \nabla \hat{f}^{(k)} / | \nabla \hat{f}^{(k)} | \right) } \cdot \left[ \frac{g}{\hat{f}^{(k)} * h} * h^* \right]$$

where $\lambda_{\text{TV}}$ is the regularization weight (controlled by `regularization_weight`, default 0.01) and the divergence-of-normalized-gradient term is the gradient of the TV penalty.

TV regularization preserves edges (such as the sharp arcs of strong lenses) while suppressing noise in flat regions. **Recommended for gravitational lens deconvolution.**

### Tikhonov regularization

Tikhonov regularization penalizes the squared $L^2$ norm of the gradient:

$$R_{\text{Tik}}(f) = \sum_{x, y} \big[ (\partial_x f)^2 + (\partial_y f)^2 \big]$$

This is computationally simpler than TV but has a known limitation: it blurs edges along with noise. Acceptable for diffuse extended sources, less so for compact features.

### No regularization

The unmodified Richardson-Lucy update is also available (`regularization: none`). Sufficient when the data is high signal-to-noise and early stopping is properly tuned.

---

## PSF Calibration and Validity

Richardson-Lucy assumes the PSF is **known and shift-invariant**. Both assumptions are imperfect in practice and merit explicit discussion.

### PSF estimation

In real telescope data, the PSF is estimated from images of stars (which are point sources at the resolution of the telescope). FLUX accepts a pre-computed PSF as a FITS or `.npy` file via the `psf_path` configuration field. The PSF must be:

- Centered (peak at the geometric center of the array)
- Normalized to unit total flux ($\sum h = 1$)
- Sized to at least $4\sigma$ on either side of the peak, where $\sigma$ is the PSF width

For DP0 simulations, the PSF is provided alongside each image set. For real data, PSF estimation is the responsibility of the user, typically using tools such as `PSFEx` or `photutils.psf`.

### Shift-invariance

The shift-invariance assumption means the PSF is the same across the entire image. This is approximately true within a single LSST detector chip but not across the full focal plane. For typical FLUX use cases (cutouts of $64 \times 64$ to $256 \times 256$ pixels around a target object), the assumption is accurate enough that the residual error is below the noise floor.

For larger fields, FLUX's deconvolver should be applied to cutouts independently, with each cutout receiving a PSF estimated at its location. This is enforced by running deconvolution **after** cutout extraction in the preprocessing chain.

---

## Numerical Implementation

The FLUX implementation in `flux/preprocessor/filters/richardson_lucy.py` follows the standard formulation with three numerical safeguards:

**Convolution in the frequency domain.** Each iteration involves two convolutions ($\hat{f} * h$ and the ratio $* h^*$). For PSFs of size $> 11 \times 11$, FFT-based convolution is faster than direct convolution. FLUX uses `scipy.signal.fftconvolve` with appropriate padding to avoid wraparound artifacts.

**Floor on the denominator.** The ratio $g / (\hat{f}^{(k)} * h)$ is undefined where the denominator is zero. FLUX adds a small floor $\epsilon = 10^{-12}$ to the denominator before division. This is large enough to prevent numerical overflow and small enough not to affect the result for any pixel with non-trivial signal.

**Boundary handling.** Convolution at image borders requires defining values outside the image. FLUX uses zero-padding with the cutout region carefully sized to leave a buffer at least equal to the PSF half-width. The cutout configuration in [modules/preprocessor.md](../modules/preprocessor.md) enforces this constraint.

The full per-iteration cost is $O(N \log N)$ where $N$ is the number of pixels, dominated by the two FFTs. For a $64 \times 64$ image with 30 iterations, this is approximately 250k operations — comfortable on any consumer GPU.

---

## Limitations

Richardson-Lucy is the most powerful filter in FLUX's pipeline, but it is not magic. Three limitations are worth stating explicitly:

**Model mismatch.** If the assumed PSF differs from the true PSF (incorrect calibration, non-shift-invariant blurring), Richardson-Lucy will produce systematic artifacts. The most common artifact is "ringing" — oscillations around bright sources. When ringing appears, suspect the PSF before suspecting the algorithm.

**Information theoretic limit.** Deconvolution cannot recover information that was destroyed by the PSF. Frequencies fully suppressed by $H(u, v) = 0$ are unrecoverable regardless of iteration count. The algorithm extrapolates from the frequencies that survived; this extrapolation is approximate and degrades with noise.

**Assumes Poisson noise.** The Bayesian derivation specifically assumes Poisson statistics. For data with significant Gaussian read noise added on top of Poisson photon noise, the multiplicative update is suboptimal but still typically usable. For data dominated by Gaussian noise alone, alternative algorithms (Wiener filtering, regularized least squares) may perform better.

For these reasons, Richardson-Lucy is a configurable option in FLUX rather than a default. Users should enable it deliberately for cases where seeing-limited resolution is the bottleneck for their downstream task.

---

## References

**Original Richardson-Lucy formulation.**
- Richardson, W. H. *Bayesian-Based Iterative Method of Image Restoration.* Journal of the Optical Society of America, 1972.
- Lucy, L. B. *An Iterative Technique for the Rectification of Observed Distributions.* The Astronomical Journal, 1974.

**EM derivation under Poisson noise.**
- Shepp, L. A. & Vardi, Y. *Maximum Likelihood Reconstruction for Emission Tomography.* IEEE Transactions on Medical Imaging, 1982.

**Total Variation regularization for Richardson-Lucy.**
- Dey, N., Blanc-Féraud, L., Zimmer, C., Roux, P., Kam, Z., Olivo-Marin, J.-C., & Zerubia, J. *Richardson-Lucy Algorithm with Total Variation Regularization for 3D Confocal Microscope Deconvolution.* Microscopy Research and Technique, 2006.

**Convergence properties and noise amplification.**
- Bertero, M. & Boccacci, P. *Introduction to Inverse Problems in Imaging.* Institute of Physics Publishing, 1998. Chapter 6 covers Richardson-Lucy convergence in detail.

**PSF estimation in astronomy.**
- Bertin, E. *Automated Morphometry with SExtractor and PSFEx.* In *Astronomical Data Analysis Software and Systems XX*, 2011.

**Modern applications and extensions in astronomy.**
- Sureau, F., Lechat, A., & Starck, J.-L. *Deep learning for a space-variant deconvolution in galaxy surveys.* Astronomy & Astrophysics, 2020. Deep-learning-based deconvolution baseline for survey-scale galaxy images.
- Akhaury, U., Jablonka, P., Starck, J.-L., et al. *Deep learning-based deconvolution for HST images.* Astronomy & Astrophysics, 2022. Application of learned deconvolution methods on Hubble Space Telescope data, demonstrating the limits of classical Richardson-Lucy.
- Michalewicz, K., et al. *STARRED: a wavelet-based two-channel deconvolution method.* Journal of Open Source Software, 2023. Modern wavelet-regularized variant of Richardson-Lucy specifically designed for astronomical light-curve extraction.
- Donath, A., et al. *Joint deconvolution of astronomical observations.* arXiv:2310.16919, 2023. Multi-image extension of Richardson-Lucy applied to combined HST and ground-based data.
- Zerbi, A., et al. *Joint multiband deconvolution for Euclid and Vera C. Rubin images.* arXiv:2502.17177, 2025. Recent Rubin-specific application directly relevant to FLUX's target use case.

**Implementation references.**

The FLUX implementation builds on `scipy.signal.fftconvolve` and `numpy` for the core operations. The TV regularization variant follows the formulation of Dey et al. (2006) cited above.

---

## See also

- [filters.md](filters.md) — the four other noise filters
- [normalization.md](normalization.md) — Z-score, min-max, and per-channel normalization
- [modules/preprocessor.md](../modules/preprocessor.md) — how Richardson-Lucy fits in the pipeline