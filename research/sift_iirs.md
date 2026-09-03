# SIFT Adaptation for ISRO Chandrayaan-2 IIRS & LRO-WAC Registration

## 1. Scale-Space Formulation & DoG Keypoint Detection

The Scale-Invariant Feature Transform (SIFT) detects scale-space extrema using Difference-of-Gaussians (DoG) functions $D(x,y,\sigma)$:

$$L(x,y,\sigma) = G(x,y,\sigma) * I(x,y)$$
$$D(x,y,\sigma) = L(x,y,k\sigma) - L(x,y,\sigma)$$

where $G(x,y,\sigma)$ is a 2D Gaussian kernel:
$$G(x,y,\sigma) = \frac{1}{2\pi \sigma^2} \exp\left(-\frac{x^2 + y^2}{2\sigma^2}\right)$$

Extrema are filtered by eliminating low-contrast candidate points ($|D(\hat{\mathbf{x}})| < 0.03$) and unstable edge responses via the Hessian matrix principal curvature ratio:
$$\frac{\text{Tr}(\mathbf{H})^2}{\text{Det}(\mathbf{H})} < \frac{(r+1)^2}{r} \quad (r = 10 \text{ default})$$

---

## 2. Radiometric Pre-processing for Cross-Sensor Matching

Due to sensor radiometric response variations between Chandrayaan-2 IIRS ($80\text{ m/px}$) and LRO WAC ($100\text{ m/px}$), patch-wise statistical normalization is performed prior to descriptor extraction:

1. **Percentile Clipping:** Intensities $I(x,y)$ are clipped at $2^{\text{nd}}$ and $98^{\text{th}}$ percentiles to eliminate extreme sensor noise:
   $$I_{\text{clip}} = \text{clamp}(I, p_2, p_{98})$$

2. **Statistic Transfer:** Source patch $I_S$ is harmonized to reference patch $I_R$:
   $$\hat{I}_S(x,y) = \frac{I_S(x,y) - \mu_S}{\sigma_S} \cdot \sigma_R + \mu_R$$

---

## 3. Algorithmic Pipeline

```
  IIRS Strip (80m) & LRO WAC Mosaic (100m)
                     │
                     ▼
  Overlapping Tiling Engine (512x512 Tiles)
                     │
                     ▼
  Patch Normalization (Percentile Clip + Stat Transfer)
                     │
                     ▼
  DoG Scale-Space Keypoint Extraction (SIFT 128-D)
                     │
                     ▼
  Lowe's Ratio Test Matching (NNDR <= 0.75)
                     │
                     ▼
  RANSAC Homography Filtering (min 8 points, tau = 3.0 px)
                     │
                     ▼
  Declustering & Z-score Outlier Rejection (Residual Z <= 2.5)
```

---

## 4. Distance Ratio & Outlier Rejection Rules

Candidate matches between 128-D descriptor vectors $\mathbf{d}_A, \mathbf{d}_B$ are validated using Lowe's Nearest Neighbor Distance Ratio (NNDR):
$$\frac{\|\mathbf{d}_A - \mathbf{d}_B^1\|_2}{\|\mathbf{d}_A - \mathbf{d}_B^2\|_2} \le 0.75$$

GCP candidates are declustered to enforce spatial spacing ($d_{\text{min}} \ge 15\text{ px}$) and filtered via residual $Z$-score thresholding:
$$Z_i = \frac{|r_i - \mu_r|}{\sigma_r} \le 2.5$$

---

## 5. Performance Metrics & Latitude Degradation

Empirical benchmarks across ~200 Chandrayaan-2 IIRS orbital strips:

| Lunar Region | Latitude Range | Solar Incidence Mismatch $\Delta \theta$ | Mean RMSE (Meters) | Mean RMSE (Pixels) |
|---|---|---|---|---|
| Equatorial | $0^\circ \text{ to } \pm 25^\circ$ | $15^\circ - 25^\circ$ | $65.4 \text{ m}$ | $0.81 \text{ px}$ |
| Mid-Latitude | $\pm 25^\circ \text{ to } \pm 50^\circ$ | $25^\circ - 45^\circ$ | $82.1 \text{ m}$ | $1.02 \text{ px}$ |
| High-Latitude | $\pm 50^\circ \text{ to } \pm 70^\circ$ | $45^\circ - 65^\circ$ | $240.5 \text{ m}$ | $3.01 \text{ px}$ |
| Polar | $>\pm 70^\circ$ | $>65^\circ$ | $>1200 \text{ m}$ | $>15.0 \text{ px}$ |

> [!CAUTION]
> **High-Latitude Breakdown:**
> SIFT gradient histograms degrade severely beyond $\pm 55^\circ$ latitude due to extreme shadow elongation and lunar surface curvature. Polar registration requires switching to RIFT or CNSFM.
