# Crater Neighborhood Structural Feature Matching (CNSFM)

## 1. Theoretical Motivation & Illumination Invariance

Conventional descriptor-based matchers (e.g., SIFT, SURF) compute spatial intensity gradients $\nabla I(x,y)$. On lunar terrain, shifts in solar illumination azimuth $\phi_{\text{sun}}$ and incidence angle $\theta_{\text{sun}}$ cause non-linear displacement of shadow boundaries.

This creates severe intensity gradient shifts:
$$\nabla I(x,y; \phi_1, \theta_1) \neq \mathbf{R}(\Delta \theta) \cdot \nabla I(x,y; \phi_2, \theta_2)$$

CNSFM sidesteps intensity degradation by transforming the correspondence domain from pixel intensity matrices $I(x,y)$ to a geometric graph of invariant topological centroids $C = \{c_i = (x_i, y_i, d_i)\}_{i=1}^N$, where $x_i, y_i$ represent crater center coordinates and $d_i$ represents crater diameter.

---

## 2. Mathematical Formulation of Invariant Descriptors

For a central crater $c_0$ and its $K$-nearest neighbor craters $\{c_1, c_2, \dots, c_K\}$, an invariant structural descriptor vector $\mathbf{v}_{c_0}$ is constructed using scale and rotation invariant geometric triples:

```
                  c_1 (x_1, y_1)
                 /  \
                /    \
               /  θ_1 \  L_{01}
              /        \
             /          \
  (x_0, y_0) c_0 ─────── c_2 (x_2, y_2)
                L_{02}
```

For each neighbor pair $(c_i, c_j)$:
1. **Distance Ratio (Scale Invariant):**
   $$r_{ij} = \frac{L_{0i}}{L_{0j}} = \frac{\|c_0 - c_i\|_2}{\|c_0 - c_j\|_2}$$

2. **Subtended Angle (Rotation Invariant):**
   $$\theta_{ij} = \arccos\left( \frac{(c_i - c_0) \cdot (c_j - c_0)}{\|c_i - c_0\|_2 \|c_j - c_0\|_2} \right)$$

3. **Relative Diameter Ratio (Scale Invariant):**
   $$s_{i} = \frac{d_i}{d_0}$$

The concatenated descriptor vector for $c_0$:
$$\mathbf{v}_{c_0} = \left[ r_{12}, r_{23}, \dots, r_{K1}, \theta_{12}, \theta_{23}, \dots, \theta_{K1}, s_1, s_2, \dots, s_K \right]^\top$$

---

## 3. Algorithmic Pipeline

```
  Input Image I_1, I_2
        │
        ▼
  YOLOv9 Crater Detector ──► Centroids C_1, C_2 & Diameters d_1, d_2
        │
        ▼
  K-NN Graph Construction ──► Form CNSF Triples (K=5)
        │
        ▼
  NNDR Metric Matching  ──► Feature Distance D(v_a, v_b) < τ * D(v_a, v_c)
        │
        ▼
  Mismatched CNSF Removal ──► Verify Local Transformation Consensus
        │
        ▼
  Output Correspondence Set {(c_i, c_j)}
```

---

## 4. Distance Metric & Matching Criteria

Matching between descriptor vectors $\mathbf{v}_a$ and $\mathbf{v}_b$ is governed by the normalized Euclidean distance:

$$D(\mathbf{v}_a, \mathbf{v}_b) = \sqrt{ \sum_{m} \left( \frac{v_{a,m} - v_{b,m}}{\sigma_m} \right)^2 }$$

A candidate match pair $(c_a, c_b)$ is accepted if it satisfies the Nearest Neighbor Distance Ratio (NNDR):
$$\frac{D(\mathbf{v}_a, \mathbf{v}_b^1)}{D(\mathbf{v}_a, \mathbf{v}_b^2)} \le \tau_{\text{NNDR}} \quad (\tau_{\text{NNDR}} \approx 0.70)$$

---

## 5. Performance Benchmarks

Empirical performance across LROC NAC lunar benchmark datasets:

| Terrain Class | Solar Azimuth Shift $\Delta \phi$ | SIFT Success % | WSSF Success % | **CNSFM Success %** | Mean Inlier Ratio |
|---|---|---|---|---|---|
| Equatorial (S1) | $12.4^\circ$ | $33.3\%$ | $100.0\%$ | **$100.0\%$** | $100.0\%$ |
| Mid-Latitude (S2) | $45.1^\circ$ | $24.4\%$ | $66.7\%$ | **$100.0\%$** | $99.3\%$ |
| South Pole (S3) | $147.7^\circ$ | $17.3\%$ | $31.2\%$ | **$72.3\%$** | $100.0\%$ |

---

## 6. Execution Constraints & Failure Modes

- **Minimum Feature Density:** Requires crater density $\rho_c \ge 5 \text{ craters per } 1024 \times 1024 \text{ tile}$.
- **Terrain Breakdown:** Fails in smooth lunar maria or young impact melt ponds where crater density $\rho_c \approx 0$.
- **Mitigation:** Gate CNSFM execution via a prior crater density check; fallback to descriptor/dense matchers (LoFTR/SuperGlue) when $\rho_c < \tau_c$.
