# Multi-Scale Fourier Phase Correlation & Sub-Pixel Peak Refinement

## 1. Theoretical Formulation

Phase correlation measures the relative spatial translation $(\Delta x, \Delta y)$ between two image signals $f_1(x,y)$ and $f_2(x,y) = f_1(x - \Delta x, y - \Delta y)$.

By the Fourier Shift Theorem:
$$F_2(u,v) = F_1(u,v) e^{-j 2\pi (u \Delta x + v \Delta y)}$$

The normalized cross-power spectrum $Q(u,v)$ isolates phase differences by factoring out magnitude variations:
$$Q(u,v) = \frac{F_1(u,v) F_2^*(u,v)}{|F_1(u,v) F_2^*(u,v)|} = e^{j 2\pi (u \Delta x + v \Delta y)}$$

Applying the inverse Fourier transform yields a Dirac delta function centered at the translation vector:
$$q(x,y) = \mathcal{F}^{-1}\{Q(u,v)\} = \delta(x - \Delta x, y - \Delta y)$$

---

## 2. Windowing & Spectral Filtering

To minimize spectral leakage induced by non-periodic image boundaries, spatial windowing functions (apodization) are evaluated prior to FFT computation:

| Window Function | Formulation $w(n), n \in [0, N-1]$ | Spectral Leakage | Sidelobe Suppression | Sub-Pixel RMSE Impact |
|---|---|---|---|---|
| **Rectangular** | $1$ | High | $-13 \text{ dB}$ | High boundary distortion |
| **Hanning** | $0.5 \left(1 - \cos\frac{2\pi n}{N-1}\right)$ | Moderate | $-31 \text{ dB}$ | Moderate attenuation |
| **Gaussian** | $\exp\left(-\frac{1}{2}\left(\frac{n - (N-1)/2}{\sigma (N-1)/2}\right)^2\right)$ | Minimal | $-42 \text{ dB}$ | **Optimal** (RMSE $0.010\text{ px}$) |
| **Tukey** | Piecewise cosine-tapered ($\alpha = 0.5$) | Low | $-35 \text{ dB}$ | Robust |
| **Blackman** | $0.42 - 0.5\cos\frac{2\pi n}{N-1} + 0.08\cos\frac{4\pi n}{N-1}$ | Severe high-freq dampening | $-58 \text{ dB}$ | **Sub-optimal** (Suppresses texture) |

---

## 3. Sub-Pixel Peak Fitting

The discrete peak $(x_0, y_0) = \arg\max_{x,y} q(x,y)$ provides integer alignment. Sub-pixel displacement $(\delta x, \delta y)$ is estimated using a 2D paraboloid surface fit over a $3 \times 3$ neighborhood around $(x_0, y_0)$:

$$q(x, y) \approx A x^2 + B y^2 + C xy + D x + E y + F$$

Solving the system of linear equations via least-squares over $q(x_0 + \Delta x, y_0 + \Delta y)$ yields the sub-pixel peak position:
$$\delta x = \frac{2 B D - C E}{C^2 - 4 A B}, \quad \delta y = \frac{2 A E - C D}{C^2 - 4 A B}$$

---

## 4. Multi-Scale Coarse-to-Fine Pipeline

```
  Level 2 (Coarse 1/4)  ──► Phase Correlation ──► Integer Translation (Δx_2, Δy_2)
                                                         │ Upsample x2
  Level 1 (Medium 1/2)  ──► Windowed Refinement  ──► Sub-Pixel Correction (Δx_1, Δy_1)
                                                         │ Upsample x2
  Level 0 (Full Res 1/1) ──► 2D Paraboloid Fit   ──► Final Vector (Δx_final, Δy_final)
```

---

## 5. Performance Benchmarks & Constraints

- **Execution Latency:** $\approx 1.2 \text{ ms}$ per $64 \times 64$ patch ($O(N^2 \log N)$ via FFTW / CuFFT).
- **Rotation Invariance:** Limited to $\pm 3^\circ$. Rotation requires conversion to Log-Polar Fourier coordinates:
  $$M(u,v) = |F(u,v)| \implies (\theta, \rho) = \left(\arctan\frac{v}{u}, \log\sqrt{u^2 + v^2}\right)$$
- **Illumination Robustness:** High tolerance to uniform gain/offset shifts due to phase-only normalization.
