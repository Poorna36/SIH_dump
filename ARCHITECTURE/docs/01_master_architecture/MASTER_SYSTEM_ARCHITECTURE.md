# Multi-Modal, Sun-Angle & Scale-Invariant Lunar Image Correspondence Suite
## Master System Architecture & Technical Specification (SIH Problem Statement 26166)

---

# 1. Executive Summary & Problem Formulation

### 1.1 Problem Statement Overview
Given high-resolution orbital imagery from Chandrayaan-2 payloads (**OHRC** $\sim 0.25\,\text{m}$, **TMC-2** $\sim 5\,\text{m}$, **IIRS** $\sim 80\,\text{m}$) and global/local lunar reference basemaps (**LRO NAC** $\sim 0.5\,\text{m}$, **LRO WAC** $\sim 100\,\text{m}$, **SELENE TC/MI**, **LOLA Geodetic Control Network**), deliver an autonomous software suite that achieves:

1. **Robust Multi-Sensor Point Correspondences** across extreme scale gaps ($\le 320\times$), arbitrary in-plane/out-of-plane rotation, perspective shear, and cross-spectral/cross-sensor radiometric discrepancies.
2. **Sub-Pixel Spatial Precision** ($\text{RMSE} < 0.5\,\text{px}$ post-refinement on equatorial imagery; $\text{RMSE} < 1.0\,\text{px}$ on rugged/polar terrain).
3. **Strict Spatial Uniformity** across heterogeneous lunar terrains (basaltic maria, crater rims, highland massifs, permanently shadowed regions).
4. **Autonomous Outlier Immunity** via multi-stage geometric filtering, eliminating dominant-plane degeneracy and local relief deformation errors.
5. **Standardized Cartographic Deliverables** including orthorectified GeoTIFFs, USGS ISIS3 cubes, tie-point matrices with full covariance matrices, and anti-circular evaluation reports.

---

### 1.2 The Four Primary Physical Challenges & Formulations

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                             THE FOUR PHYSICAL CHALLENGES                                 │
├───────────────────────┬───────────────────────────────────┬──────────────────────────────┤
│ Challenge             │ Physical / Mathematical Cause     │ Architectural Solution       │
├───────────────────────┼───────────────────────────────────┼──────────────────────────────┤
│ 1. Illumination &     │ Grazing incidence angles create   │ • Frequency Phase Congruency │
│    Sun-Angle Shift    │ dynamic shadow boundary motion;   │ • GNN Cross-Attention        │
│                       │ pixel intensity gradients invert  │ • Topological Crater Graphs  │
│                       │ and decorrelate completely.       │   (CNSFM Geometry-Only)      │
├───────────────────────┼───────────────────────────────────┼──────────────────────────────┤
│ 2. Extreme Scale      │ 320× GSD ratio (OHRC 0.25m vs     │ • Transitive Registration    │
│    Discrepancy        │ IIRS 80m) destroys local pixel    │   (OHRC → NAC → WAC → IIRS)  │
│                       │ texture continuity and high-freq  │ • Multi-Scale Log-Gabor      │
│                       │ structural correspondence.        │   Scale-Space Resampling     │
├───────────────────────┼───────────────────────────────────┼──────────────────────────────┤
│ 3. Multi-Modal /      │ Cross-spectral reflectance shifts │ • Maximum Index Map (MIM)    │
│    Radiometric Shift  │ (panchromatic vs 250-band hypersp │ • Deep Context Matching      │
│                       │ vs SAR DFSAR-SELENE dielectric)   │ • Percentile-Clip + Stat     │
│                       │ break intensity constancy.        │   Transfer Normalization     │
├───────────────────────┼───────────────────────────────────┼──────────────────────────────┤
│ 4. Planar Degeneracy  │ Flat lunar maria trigger standard │ • DEGENSAC Degeneracy Guard  │
│    & Rugged Relief    │ RANSAC collapse; steep crater rim │ • Terrain-Adaptive Models    │
│    Parallax           │ parallax creates non-projective   │   (Affine / Homog / TPS Mesh)│
│                       │ piecewise local deformation.      │ • Post-RANSAC Z-Score Filter │
└───────────────────────┴───────────────────────────────────┴──────────────────────────────┘
```

---

# 2. Master System Pipeline Architecture

The system operates as a coarse-to-fine hierarchical cascade:

```
╔══════════════════════════════════════════════════════════════════════════════════════════╗
║                               STAGE 0: DATA INGESTION                                    ║
║  USGS ISIS3 (isisimport, spiceinit, cam2map) → NASA ASP (mapproject, ortho)             ║
║  Extract Metadata: Corner Lat/Lon, Solar Incidence (i), Azimuth (φ), GSD, Sensor ID      ║
╚══════════════════════════════════════════════╦═══════════════════════════════════════════╝
                                               ║
                                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 1: METADATA-DRIVEN SPATIAL NARROWING & ROI EXTRACTION                             ┃
┃  ───────────────────────────────────────────────────────────                             ┃
┃  • Compute Geodetic Bounding Box with Ephemeris Uncertainty Expansion:                   ┃
┃    ΔDeg = (2.5 · σ_ephemeris) / (R_Moon · π / 180°)                                      ┃
┃  • Query NASA Moon Trek WMTS API (trek.nasa.gov) OR Crop Local Reference VRT/GeoTIFF     ┃
┃  • Compute Solar Azimuth Difference: Δφ = |φ_src - φ_ref|                                ┃
┃  • Compute Scale Ratio: S_ratio = GSD_coarse / GSD_fine                                  ┃
┃  • Classify Regime: NORMAL (Δφ ≤ 60°) │ EXTREME_ILLUM (Δφ > 60° / Polar) │ EXTREME_SCALE ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                               ║
                                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 2: PREPROCESSING & INTENSITY HARMONIZATION                                        ┃
┃  ────────────────────────────────────────────────                                        ┃
┃  • Scale Harmonization: Resample coarse image to target GSD                              ┃
┃    - IF Solar Elevation > 30°: Bicubic Interpolation (high-frequency preservation)       ┃
┃    - IF Solar Elevation ≤ 30°: Bilinear Interpolation (prevents shadow-edge ringing)     ┃
┃  • Branch A (Panchromatic): 2nd/98th Percentile Clip → Stat Transfer → 8-bit uint8      ┃
┃  • Branch B (Cross-Modal / IIRS): Band Selection (0.85-1.05 μm) → Lommel-Seeliger Photom ┃
┃  • Branch C (Polar Shadow): Dynamic Range Percentile Stretch + Bilateral Edge Filter     ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                               ║
                       ┌───────────────────────╨───────────────────────┐
                       │           TERRAIN-ADAPTIVE ROUTER             │
                       │   Inputs: Sensor Pair, Δφ, Latitude, Terrain  │
                       └───────┬───────────────────────┬───────────────┘
                               │                       │
           NORMAL / CROSS-MODAL│                       │ EXTREME ILLUMINATION
           (Equatorial / Mod)  │                       │ (Polar / Δφ > 60°)
                               ▼                       ▼
            ┌─────────────────────────────┐ ┌─────────────────────────────┐
            │ STAGE 3A: PRIMARY MATCHER   │ │ STAGE 3B: TOPOLOGICAL PATH  │
            │ SuperPoint + Supplementary  │ │ YOLOv9 / DeepMoon Crater    │
            │ Phase Congruency Keypoints  │ │ Detection → CNSFM Geometric │
            │ → LightGlue Attention GNN   │ │ Triplets (α, ρ, γ) Invariant│
            │ → Dustbin Filter (τ > 0.2)  │ │ → MCR Outlier Pruning       │
            │ → Spatial Domain Bounds Chk │ │ (72.3% Polar Success Rate)  │
            └──────────────┬──────────────┘ └──────────────┬──────────────┘
                           │                               │
                           │   ┌───────────────────────────┘
                           │   │  [Fallback: If craters < 10 or inliers < 15,
                           │   │   route dynamically between 3A and 3B]
                           ▼   ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 4: SPATIAL UNIFORMITY ENFORCEMENT (ANMS)                                          ┃
┃  ──────────────────────────────────────────────                                          ┃
┃  • Suppress-via-Size-Clustered (SSC) Algorithm: Adaptive Suppression Radius r_i          ┃
┃  • Enforce Keypoint Budget N_max = 500 across 8×8 Spatial Tile Grid                      ┃
┃  • Compute Spatial Shannon Entropy: H_spatial = -Σ p_k ln(p_k) / ln(M)                   ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                               ║
                                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 5: THREE-STAGE ROBUST GEOMETRIC SOLVER                                            ┃
┃  ────────────────────────────────────────────                                            ┃
┃  • Stage 5.1 (Learned Pre-Filter): LightGlue Dustbin Filtering (p_dustbin < 0.8)         ┃
┃  • Stage 5.2 (Data-Seeded DEGENSAC):                                                     │
┃    - Seeded from Top-10 Mutual Confidence Matches (prevents DE divergence)               │
┃    - Flat Mare (< 15° slope)      → Affine 6-DOF + DEGENSAC (0.5 px threshold)           │
┃    - Oblique / Moderate Relief     → Homography 8-DOF + DEGENSAC                         │
┃    - Rugged Highland (> 30° slope) → Piecewise Affine Thin-Plate Spline (TPS Mesh)       │
┃  • Stage 5.3 (Post-RANSAC Statistical Cleaning):                                         │
┃    - Spatial GCP Declustering (min 15 px inter-point distance)                           │
┃    - Residual Z-Score Outlier Rejection (|z| > 2.5σ pruned for N > 20)                   │
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                               ║
                                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 6: ANALYTICAL SUB-PIXEL REFINEMENT & ERROR PROPAGATION                            ┃
┃  ────────────────────────────────────────────────────────────                            ┃
┃  • Per-Match Local Patch Extraction (64×64 px) with 2D Gaussian Apodization Window        ┃
┃  • Phase-Only Cross-Power Spectrum: R(u,v) = (F1 · F2*) / |F1 · F2*|                     ┃
┃  • 2D Paraboloid Peak Fitting on 3×3 Neighborhood → Sub-pixel offset (δx, δy)             ┃
┃  • Gauss-Helmert Covariance Propagation → Formal Point Standard Errors (σ_x, σ_y)        ┃
┃  • Quality Gate: Reject if ||δx, δy|| > 2.0 px OR Correlation Peak < 0.35                ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
                                               ║
                                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 7: CARTOGRAPHIC PRODUCT GENERATION & ANTI-CIRCULAR EVALUATION                     ┃
┃  ───────────────────────────────────────────────────────────────────                     ┃
┃  • Final Transformation Warp to Reference CRS (Selenographic Equirectangular / EPSG:104903)┃
┃  • Dual Product Generation: Cloud-Optimized GeoTIFF (.tif) + Standard USGS ISIS (.cub)   ┃
┃  • Export Full Tie-Point Matrix CSV + JSON Transform Parameter File                      ┃
┃  • 20% Held-Out Validation Protocol: Calculate Checkpoint RMSE_x, RMSE_y, RMSE_total     ┃
┃  • Metrics: MedAE, Sub-Pixel Pct (<0.5 px, <1.0 px), Inlier Ratio, Spatial Uniformity U   ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
```

---

# 3. Component Classification & Decision Records

### 3.1 Component Classification Matrix

| Component | Architecture Tier | Rationale & Evidence | Failure Impact If Omitted |
|:---|:---:|:---|:---|
| **Stage 1: Ephemeris Spatial Narrowing** | **ESSENTIAL** | Reduces planet-wide search to bounded ROI. Demonstrated in [MoonMetaSync] on OHRC↔TMC-2. | Computational collapse; massive combinatorial false match rate. |
| **Stage 2: Percentile Intensity Normalization** | **ESSENTIAL** | Harmonizes dynamic range across disparate sensors. Validated on IIRS [SIFT-IIRS-WAC]. | Cross-sensor feature descriptor divergence; learned matching fails. |
| **Stage 3A: LightGlue + SuperPoint** | **ESSENTIAL** | Only learned matcher with 6/6 condition success on Chandrayaan-2 payloads [Traditional_vs_DL]. | Failure on polar and cross-modal pairs; degrades to 3-6 px RMSE. |
| **Stage 3A: Supplementary PC Keypoints** | **HIGH-VALUE** | Phase congruency keypoints capture structures where intensity gradients vanish. | Reduced keypoint density in low-texture mare and extreme shadow. |
| **Stage 3B: CNSFM Crater Geometry** | **HIGH-VALUE** | 72.3% polar success rate under 147.7° azimuth diff vs 31.2% for all alternatives [CNSFM]. | Total pipeline failure on extreme polar shadow inversion pairs. |
| **Stage 4: ANMS-SSC Spatial Uniformity** | **ESSENTIAL** | Fulfills explicit mandate for uniform match distribution; O(n) complexity [ANMS-Codes]. | Point clustering around high-contrast crater rims; unconstrained warp. |
| **Stage 5.2: DEGENSAC Robust Estimator** | **ESSENTIAL** | Guards against dominant coplanar degenerate sample selection on flat lunar maria. | Degenerate projective homographies; geometric instability. |
| **Stage 5.2: Terrain-Adaptive TPS Warp** | **HIGH-VALUE** | Captures local non-rigid parallax on crater rims and mountains [DESCA, SIFT-IIRS-WAC]. | Global affine/homography models shear and fail on highland terrain. |
| **Stage 5.3: Post-RANSAC Z-Score Filter** | **HIGH-VALUE** | Removes gross residual errors surviving RANSAC consensus [SIFT-IIRS-WAC]. | Outlier contamination degrades sub-pixel registration accuracy. |
| **Stage 6: Paraboloid Phase Refinement** | **ESSENTIAL** | Bridges 0.5-0.9 px matcher error down to < 0.2 px analytical precision [HybridPhaseCorrelation]. | System fails explicit sub-pixel accuracy requirement. |
| **Stage 7: Anti-Circularity Protocol** | **ESSENTIAL** | Evaluates true generalization error on 20% held-out checkpoints (not training points). | Overfitting; scientifically invalid circular error reporting. |

---

### 3.2 Formal Decision Records

#### Decision 1: Primary Matcher Selection — LightGlue vs. LNIFT vs. RIFT2
- **Chosen:** **LightGlue (with SuperPoint front-end)** as Primary Workhorse.
- **Decision Rationale:** Direct empirical benchmarks on Chandrayaan-2 OHRC, TMC-2, IIRS, and DFSAR [Traditional_vs_DL] prove LightGlue/SuperGlue is the **only** method that achieves 100% test matrix convergence with lowest RMSE ($0.4\text{--}0.9\,\text{px}$). 
- **Why LNIFT is Positioned as a Baseline/Fallback:** LNIFT offers compelling speed ($0.49\,\text{s}$ vs $47.8\,\text{s}$ for RIFT) and built-in ANMS, but has **zero published validation on lunar polar terrain or Chandrayaan-2 payloads**. Accuracy-first architecture mandates prioritizing flight-validated methods while retaining LNIFT as a high-speed CPU alternative.

#### Decision 2: Extreme Illumination Solver — CNSFM vs. Appearance Descriptors
- **Chosen:** **CNSFM (Crater Neighborhood Signature Feature Matching)** with MCR Outlier Filtering.
- **Decision Rationale:** When solar azimuth difference $\Delta\varphi > 90^\circ$, shadow inversion causes 100% of pixel-intensity-based descriptors (SIFT, SURF, ORB) to fail. CNSFM matches topological graphs of crater centers and radii $(\alpha, \rho, \gamma)$—which are **mathematically invariant to illumination by construction**.

#### Decision 3: Sub-Pixel Refinement — Phase Correlation vs. MI-DE Optimization
- **Chosen:** **Gaussian-Windowed 2D Paraboloid Peak Fitting Phase Correlation**.
- **Decision Rationale:** Benchmarks across 7 sub-pixel algorithms in [HybridPhaseCorrelation] prove 2D paraboloid fitting delivers $\text{RMSE} = 0.010\,\text{px}$ (96.3% improvement over integer matching) at $< 1\,\text{ms}$ per point, whereas Mutual Information + Differential Evolution (MI-DE) requires $\sim 30\,\text{s}$ of stochastic search with lower stability.

---

# 4. Sensor-Pair Profiles & Transitive Chaining Strategy

The system configures operational hyperparameters based on sensor physical geometry:

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                         SENSOR-PAIR SPECIFICATION MATRIX                                 │
├───────────────────┬──────────────┬──────────────┬───────────────────┬────────────────────┤
│ Sensor Pair       │ Scale Ratio  │ Primary Risk │ Routing Pipeline  │ Expected Target    │
├───────────────────┼──────────────┼──────────────┼───────────────────┼────────────────────┤
│ OHRC ↔ LRO NAC    │ 0.5× – 2.0×  │ Illumination │ Stage 3A (Primary)│ RMSE < 0.35 px     │
│ (0.25m vs 0.5m)   │ (Minimal)    │ & Shadows    │ Stage 3B (Polar)  │ Inliers > 200      │
├───────────────────┼──────────────┼──────────────┼───────────────────┼────────────────────┤
│ TMC-2 ↔ LRO WAC   │ 15× – 20×    │ Resolution   │ Stage 3A with GSD │ RMSE < 1.0 px      │
│ (5.0m vs 100m)    │ (Moderate)   │ Gap + Albedo │ Pre-Resampling    │ Inliers > 80       │
├───────────────────┼──────────────┼──────────────┼───────────────────┼────────────────────┤
│ OHRC ↔ TMC-2      │ 20×          │ Scale Gap    │ Stage 3A + Bilin/ │ RMSE < 0.5 px      │
│ (0.25m vs 5.0m)   │ (Intra-Craft)│ (Same Solar) │ Bicubic Resample  │ Inliers > 150      │
├───────────────────┼──────────────┼──────────────┼───────────────────┼────────────────────┤
│ IIRS ↔ LRO WAC    │ 1.25×        │ Hyperspectral│ Stage 2 Branch B  │ RMSE < 0.9 px      │
│ (80m vs 100m)     │ (Minimal)    │ Modality Gap │ → Stage 3A (NIR)  │ (~73m ground acc.) │
├───────────────────┼──────────────┼──────────────┼───────────────────┼────────────────────┤
│ OHRC ↔ IIRS       │ 320×         │ Extreme Scale│ TRANSITIVE CHAIN  │ RMSE < 2.5 px      │
│ (0.25m vs 80m)    │ (Intractable)│ + Modality   │ (See §4.1 below)  │ (in IIRS grid)     │
└───────────────────┴──────────────┴──────────────┴───────────────────┴────────────────────┘
```

---

### 4.1 The 320× Transitive Registration Chain

Direct correspondence between 0.25m OHRC and 80m IIRS is mathematically ill-posed due to sub-resolution texture destruction. The pipeline resolves this via **Transitive Control Chaining**:

```
      OHRC (0.25m)
           │
           │  Direct Registration (Stage 3A: Scale ~2×)
           ▼
      LRO NAC (0.5m)
           │
           │  Pre-Registered LRO Geodetic Control Network
           ▼
      LRO WAC (100m)
           ▲
           │  Direct Cross-Spectral Registration (Stage 2 Branch B: Scale ~1.25×)
           │
      IIRS (80m)
```

$$\mathbf{T}_{\text{OHRC}\rightarrow\text{IIRS}} = \mathbf{T}_{\text{WAC}\rightarrow\text{IIRS}} \cdot \mathbf{T}_{\text{NAC}\rightarrow\text{WAC}} \cdot \mathbf{T}_{\text{OHRC}\rightarrow\text{NAC}}$$

This converts an intractable single-step problem ($320\times$) into two well-conditioned sub-problems ($2\times$ and $1.25\times$), bound by the flight-proven LRO LOLA-tied geodetic frame.

---

# 5. Rigorous Mathematical Formulations by Stage

### Stage 1: Geodetic Bounding Box Expansion
Given sensor ephemeris pointing uncertainty $\sigma_{\text{ephemeris}}$:
$$\Delta\text{Lat} = \frac{k \cdot \sigma_{\text{ephemeris}}}{R_{\text{Moon}} \cdot \frac{\pi}{180^\circ}}, \quad \Delta\text{Lon} = \frac{k \cdot \sigma_{\text{ephemeris}}}{R_{\text{Moon}} \cos(\text{Lat}_{\text{center}}) \cdot \frac{\pi}{180^\circ}} \quad (k=2.5, R_{\text{Moon}} = 1737.4\,\text{km})$$

---

### Stage 2: Log-Gabor Phase Congruency & Maximum Index Map (MIM)
The 2D Log-Gabor transfer function in polar frequency coordinates $(\omega, \theta)$:
$$G_{s,o}(\omega, \theta) = \exp\left( -\frac{(\ln(\omega/\omega_{s0}))^2}{2(\ln(\kappa_\omega))^2} \right) \cdot \exp\left( -\frac{(\theta - \theta_o)^2}{2\sigma_\theta^2} \right)$$
Where $s \in \{1,\dots,N_s\}$ (scales, $N_s=4$) and $o \in \{1,\dots,N_o\}$ (orientations, $N_o=6$).

The Phase Congruency energy $PC(x,y)$ computes local phase alignment:
$$PC(x,y) = \frac{\sum_o \sum_s W_o(x,y) \lfloor A_{s,o}(x,y) \Delta\Phi_{s,o}(x,y) - T_o \rfloor_+}{\sum_o \sum_s A_{s,o}(x,y) + \epsilon}$$

The **Maximum Index Map (MIM)** descriptor extracts the dominant directional frequency response:
$$\text{MIM}(x,y) = \arg\max_{o \in \{1,\dots,N_o\}} \left( \sum_{s=1}^{N_s} A_{s,o}(x,y) \right)$$

---

### Stage 3B: CNSFM Similarity Invariant Topologies
For detected crater centers $C_i = (x_i, y_i)$ and radii $r_i$, local $K$-nearest neighbor triples $(C_0, C_i, C_j)$ form similarity-invariant geometric descriptors:
$$\alpha = \arccos\left( \frac{\vec{v}_{0i} \cdot \vec{v}_{0j}}{\|\vec{v}_{0i}\| \|\vec{v}_{0j}\|} \right), \quad \rho = \frac{\|C_0 - C_i\|}{\|C_0 - C_j\|}, \quad \gamma = \frac{r_i}{r_j}$$

The geometric distance between candidate triples $T_a$ and $T_b$:
$$D(T_a, T_b) = w_\alpha |\alpha_a - \alpha_b| + w_\rho |\rho_a - \rho_b| + w_\gamma |\gamma_a - \gamma_b|$$

---

### Stage 4: Spatial Uniformity via ANMS-SSC
For keypoint set $\mathcal{P} = \{\mathbf{p}_i\}_{i=1}^N$ sorted by confidence score $c_i$ in descending order, the suppression radius $r_i$ is defined as:
$$r_i = \min_{j: c_j > c_i} \|\mathbf{p}_i - \mathbf{p}_j\|_2$$
ANMS selects the top $N_{\text{max}} = 500$ points maximizing local suppression radius. The **Spatial Shannon Uniformity Index** $U \in [0, 1]$ over an $M$-cell grid ($M = 8 \times 8 = 64$):
$$U = \frac{-\sum_{k=1}^M p_k \ln(p_k)}{\ln(M)}, \quad p_k = \frac{n_k}{\sum_{j=1}^M n_j}$$
*(Where $n_k$ is the match count in cell $k$; $U = 1.0$ represents perfect uniform spatial entropy).*

---

### Stage 5: Terrain-Adaptive Deformation Models

#### Model A: Affine (6-DOF) — Flat Basaltic Maria
$$\begin{bmatrix} x_{\text{ref}} \\ y_{\text{ref}} \end{bmatrix} = \begin{bmatrix} a_{11} & a_{12} \\ a_{21} & a_{22} \end{bmatrix} \begin{bmatrix} x_{\text{src}} \\ y_{\text{src}} \end{bmatrix} + \begin{bmatrix} t_x \\ t_y \end{bmatrix}$$

#### Model B: Projective Homography (8-DOF) — Oblique Viewing Geometry
$$\begin{bmatrix} x_{\text{ref}}' \\ y_{\text{ref}}' \\ w' \end{bmatrix} = \begin{bmatrix} h_{11} & h_{12} & h_{13} \\ h_{21} & h_{22} & h_{23} \\ h_{31} & h_{32} & 1 \end{bmatrix} \begin{bmatrix} x_{\text{src}} \\ y_{\text{src}} \\ 1 \end{bmatrix}, \quad x_{\text{ref}} = \frac{x_{\text{ref}}'}{w'}, \quad y_{\text{ref}} = \frac{y_{\text{ref}}'}{w'}$$

#### Model C: Thin-Plate Spline (TPS) — Rugged Highland Terrain
$$f(x,y) = a_0 + a_x x + a_y y + \sum_{i=1}^K w_i U(\|\mathbf{p} - \mathbf{p}_i\|), \quad U(r) = r^2 \ln(r)$$

---

### Stage 6: Analytical Sub-Pixel Phase Paraboloid Fitting
For local $64 \times 64$ patches $f(x,y)$ and $g(x,y)$ apodized by 2D Gaussian window $W(x,y) = \exp\left(-\frac{(x^2+y^2)}{2\sigma^2}\right)$:
$$R(u,v) = \frac{\mathcal{F}\{f \cdot W\} \cdot \mathcal{F}^*\{g \cdot W\}}{|\mathcal{F}\{f \cdot W\} \cdot \mathcal{F}^*\{g \cdot W\}| + \epsilon}, \quad r(x,y) = \mathcal{F}^{-1}\{R(u,v)\}$$

Around the discrete peak $(x_0, y_0) = \arg\max r(x,y)$, fit a continuous 2D paraboloid surface:
$$P(x,y) = A x^2 + B y^2 + C x y + D x + E y + F$$
Solving the linear least-squares system over the $3 \times 3$ neighborhood yields analytical vertex offsets:
$$\delta x = \frac{2 B D - C E}{C^2 - 4 A B}, \quad \delta y = \frac{2 A E - C D}{C^2 - 4 A B}$$
The Gauss-Helmert covariance propagation yields point variance:
$$\sigma_x^2 = \mathbf{J}_x \mathbf{\Sigma}_r \mathbf{J}_x^T, \quad \sigma_y^2 = \mathbf{J}_y \mathbf{\Sigma}_r \mathbf{J}_y^T$$

---

# 6. Detailed Implementation Algorithms & Pseudocode

### 6.1 Complete End-to-End Orchestrator

```python
"""
SIH 26166 Master Registration Orchestrator
Production-Grade Execution Engine
"""
import numpy as np
import cv2
import torch
import pydegensac
import rasterio
from dataclasses import dataclass
from typing import Dict, List, Tuple, Optional

@dataclass
class RegistrationConfig:
    min_inliers: int = 30
    min_inlier_ratio: float = 0.15
    dustbin_threshold: float = 0.20
    anms_target_points: int = 500
    subpixel_patch_size: int = 64
    z_score_threshold: float = 2.5
    grid_size: int = 8

class MasterRegistrationPipeline:
    def __init__(self, config: RegistrationConfig = RegistrationConfig()):
        self.cfg = config
        self.device = 'cuda' if torch.cuda.is_available() else 'cpu'
        self._init_models()

    def _init_models(self):
        # Load LightGlue with SuperPoint front-end via PyTorch
        from lightglue import LightGlue, SuperPoint
        self.extractor = SuperPoint(max_num_keypoints=2048).eval().to(self.device)
        self.matcher = LightGlue(features='superpoint', depth_confidence=0.95).eval().to(self.device)

    def execute(self, source_path: str, reference_path: str, metadata: dict) -> dict:
        # Step 1: Spatial Narrowing & Ephemeris Crop
        src_patch, ref_patch = self._extract_roi(source_path, reference_path, metadata)
        
        # Step 2: Adaptive Resampling & Normalization
        src_norm, ref_norm = self._preprocess_pair(src_patch, ref_patch, metadata)
        
        # Step 3: Routed Feature Matching
        matches = self._match_routed(src_norm, ref_norm, metadata)
        
        # Step 4: Spatial Uniformity Enforcement (ANMS-SSC)
        uniform_matches, uniformity_index = self._enforce_uniformity(matches, src_norm.shape)
        
        # Step 5: Three-Stage Robust Solver (DEGENSAC + Model Selection + Z-Score)
        inliers, transform_model = self._solve_geometry(uniform_matches, metadata)
        
        # Step 6: Analytical Sub-Pixel Refinement
        refined_inliers = self._subpixel_refine(src_norm, ref_norm, inliers)
        
        # Step 7: Anti-Circularity Validation & Product Generation
        evaluation_report = self._evaluate_held_out(refined_inliers, transform_model)
        registered_product = self._generate_products(src_patch, transform_model, metadata)
        
        return {
            "correspondences": refined_inliers,
            "transform": transform_model,
            "metrics": evaluation_report,
            "spatial_uniformity_score": uniformity_index,
            "registered_image": registered_product
        }

    def _preprocess_pair(self, src: np.ndarray, ref: np.ndarray, meta: dict) -> Tuple[np.ndarray, np.ndarray]:
        # Sun-angle adaptive interpolation selection
        interp = cv2.INTER_CUBIC if meta.get('sun_elevation', 45) > 30 else cv2.INTER_LINEAR
        if meta.get('resample_ratio', 1.0) != 1.0:
            target_dim = (int(ref.shape[1]), int(ref.shape[0]))
            src = cv2.resize(src, target_dim, interpolation=interp)
            
        # Percentile-clip dynamic range harmonization (2nd/98th percentile)
        p2, p98 = np.percentile(src, (2, 98))
        src_clipped = np.clip((src - p2) / (p98 - p2 + 1e-7) * 255.0, 0, 255).astype(np.uint8)
        
        rp2, rp98 = np.percentile(ref, (2, 98))
        ref_clipped = np.clip((ref - rp2) / (rp98 - rp2 + 1e-7) * 255.0, 0, 255).astype(np.uint8)
        
        return src_clipped, ref_clipped

    def _subpixel_refine(self, src: np.ndarray, ref: np.ndarray, inliers: List[dict]) -> List[dict]:
        refined = []
        half = self.cfg.subpixel_patch_size // 2
        # Precompute 2D Gaussian Apodization Window
        w1d = cv2.getGaussianKernel(self.cfg.subpixel_patch_size, self.cfg.subpixel_patch_size / 6)
        window = np.outer(w1d, w1d)

        for match in inliers:
            x_s, y_s = int(round(match['x_src'])), int(round(match['y_src']))
            x_r, y_r = int(round(match['x_ref'])), int(round(match['y_ref']))

            if (y_s - half < 0 or y_s + half >= src.shape[0] or 
                x_s - half < 0 or x_s + half >= src.shape[1] or
                y_r - half < 0 or y_r + half >= ref.shape[0] or 
                x_r - half < 0 or x_r + half >= ref.shape[1]):
                refined.append(match)
                continue

            patch_s = src[y_s - half:y_s + half, x_s - half:x_s + half].astype(np.float64) * window
            patch_r = ref[y_r - half:y_r + half, x_r - half:x_r + half].astype(np.float64) * window

            # Phase-Only Cross-Power Spectrum
            F1 = np.fft.fft2(patch_s)
            F2 = np.fft.fft2(patch_r)
            denom = np.abs(F1 * np.conj(F2)) + 1e-15
            R = (F1 * np.conj(F2)) / denom
            corr = np.real(np.fft.ifft2(R))

            py, px = np.unravel_index(np.argmax(corr), corr.shape)
            peak_val = corr[py, px]

            # Fit 2D Paraboloid over 3x3 window if peak is central
            if 1 <= px < corr.shape[1] - 1 and 1 <= py < corr.shape[0] - 1 and peak_val > 0.35:
                dx = 0.5 * (corr[py, px + 1] - corr[py, px - 1]) / (2 * corr[py, px] - corr[py, px + 1] - corr[py, px - 1] + 1e-12)
                dy = 0.5 * (corr[py + 1, px] - corr[py - 1, px]) / (2 * corr[py, px] - corr[py + 1, px] - corr[py - 1, px] + 1e-12)
                
                if np.sqrt(dx**2 + dy**2) < 2.0:
                    match['x_ref_subpx'] = match['x_ref'] + dx
                    match['y_ref_subpx'] = match['y_ref'] + dy
                    match['is_refined'] = True
                    match['corr_peak'] = float(peak_val)
                    refined.append(match)
                    continue

            match['is_refined'] = False
            refined.append(match)

        return refined
```

---

# 7. Comprehensive Anti-Patterns & Excluded Methods

The system explicitly excludes the following approaches based on definitive negative evidence in literature:

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                      EVIDENCE-BACKED ANTI-PATTERNS & EXCLUSIONS                          │
├───────────────────────────────┬───────────────────────────────┬──────────────────────────┤
│ Excluded Approach             │ Demonstrated Failure Mode     │ Evidence & Reference     │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ SIFT as Standalone Matcher    │ 0% Success Rate at Lunar      │ [Traditional_vs_DL],     │
│                               │ Poles; shadow inversion kills │ [CNSFM], [MoonMetaSync]  │
│                               │ gradient orientation bins.    │                          │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ Heavy Preprocessing Cascades  │ Adding CLAHE + Inversion +    │ [Traditional_vs_DL §P2]  │
│ (CLAHE + Morph + PCA)         │ Dilation fails to rescue SIFT │                          │
│                               │ and degrades learned matchers.│                          │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ Naive Descriptor Fusion       │ IntFeat (SIFT+ORB) descriptor │ [MoonMetaSync §P4]       │
│ (IntFeat / SIFT+ORB)          │ concatenation amplifies noise │                          │
│                               │ without improving accuracy.   │                          │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ Single Global Affine Warp     │ Catastrophic model failure on │ [DESCA §P4],             │
│                               │ highland relief & crater rims │ [HybridPhaseCorrelation] │
│                               │ due to local parallax shear.  │                          │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ Random-Initialized DE         │ Random search initialization  │ [DESCA §P2]              │
│ (Differential Evolution)      │ causes divergence to 0 inliers│                          │
│                               │ (must seed from top matches). │                          │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ Blackman Apodization Window   │ Suppresses high-frequency     │ [HybridPhaseCorrelation  │
│ in Phase Correlation          │ spectral energy, degrading    │  §P4]                    │
│                               │ sub-pixel accuracy.           │                          │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ Unchecked LoFTR Inference     │ Produces out-of-image-domain  │ [HybridPhaseCorrelation  │
│                               │ extrapolated correspondences. │  §P4]                    │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ SSIM / PSNR as Primary Metric │ Measures visual pixel albedo, │ [HybridPhaseCorrelation  │
│                               │ not true geometric alignment. │  §P0]                    │
├───────────────────────────────┼───────────────────────────────┼──────────────────────────┤
│ cGAN Radiometric Normalization│ Assumes pre-aligned images;   │ [Radiometric_Normaliz.]  │
│                               │ hallucinates terrain features.│                          │
└───────────────────────────────┴───────────────────────────────┴──────────────────────────┘
```

---

# 8. Rigorous Evaluation Protocol & Ablation Study

### 8.1 The 20% Held-Out Checkpoint Validation (Anti-Circularity)
To guarantee that reported RMSE reflects true geodetic generalization rather than over-parameterized model memorization:
1. Divide verified inlier correspondences into an **80% Estimation Set** and **20% Held-Out Checkpoint Set**.
2. Fit the selected transformation model $\mathcal{M}$ exclusively on the 80% partition.
3. Compute the final evaluation metrics **strictly on the 20% held-out checkpoints**:

$$\text{RMSE}_{\text{total}} = \sqrt{\frac{1}{N_{\text{val}}} \sum_{i=1}^{N_{\text{val}}} \|\mathcal{M}(\mathbf{p}_{i,\text{src}}) - \mathbf{p}_{i,\text{ref}}\|^2_2}$$

$$\text{MedAE} = \text{median}\left( \|\mathcal{M}(\mathbf{p}_{i,\text{src}}) - \mathbf{p}_{i,\text{ref}}\|_2 \right)$$

$$\text{SubPixel}_{0.5} = \frac{1}{N_{\text{val}}} \sum_{i=1}^{N_{\text{val}}} \mathbb{I}\left( \|\mathcal{M}(\mathbf{p}_{i,\text{src}}) - \mathbf{p}_{i,\text{ref}}\|_2 < 0.5\,\text{px} \right) \times 100\%$$

---

### 8.2 Comprehensive 10-Configuration Ablation Matrix

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                             FULL ABLATION EXPERIMENT MATRIX                              │
├────┬───────────────────────────────┬───────────────────────────────┬─────────────────────┤
│ #  │ Configuration                 │ Component Modified / Removed  │ Target Metric Tested│
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 1  │ **Full Pipeline (Master)**    │ None (All components enabled) │ Baseline Benchmark  │
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 2  │ Ablate CNSFM (Stage 3B)       │ Remove Crater Geometry Path   │ Polar Success Rate  │
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 3  │ Ablate PC Keypoints (Stage 3A)│ Remove Phase Congruency Feats │ Low-Texture Inliers │
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 4  │ Ablate ANMS-SSC (Stage 4)     │ Replace with Random Top-K     │ Spatial Entropy (U) │
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 5  │ Ablate DEGENSAC (Stage 5.2)   │ Replace with Plain RANSAC     │ Mare Degeneracy Rate│
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 6  │ Ablate Sub-Pixel Refinement   │ Integer Match Output Only     │ Checkpoint RMSE     │
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 7  │ Ablate Z-Score Filter (5.3)   │ Two-Stage Filter Only         │ Max Error Outliers  │
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 8  │ Replace LightGlue with RIFT2  │ Classical Frequency Matcher   │ Cross-Modal RMSE    │
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 9  │ Replace LightGlue with LNIFT  │ High-Speed Spatial Matcher    │ Latency vs RMSE     │
├────┼───────────────────────────────┼───────────────────────────────┼─────────────────────┤
│ 10 │ Ablate TPS Mesh Transform     │ Single Global Affine on Mtn   │ Highland Relief RMSE│
└────┴───────────────────────────────┴───────────────────────────────┴─────────────────────┘
```

---

### 8.3 The 11 Test Benchmark Categories

1. **`EQ-SS` (Equatorial Same-Sensor)**: OHRC↔OHRC, TMC↔TMC ($0^\circ\text{--}30^\circ$ latitude).
2. **`EQ-XS` (Equatorial Cross-Sensor)**: OHRC↔LRO NAC (moderate scale ratio $\sim 2\times$).
3. **`EQ-XS-WIDE` (Wide Scale Gap)**: OHRC↔TMC-2 ($20\times$ scale ratio).
4. **`POL-SS` (Polar Same-Sensor)**: OHRC↔OHRC polar ($> 75^\circ$ latitude).
5. **`POL-XS` (Polar Cross-Sensor)**: OHRC↔LRO NAC polar (extreme illumination variation).
6. **`SUN-LOW` (Grazing Sun Elevation)**: Solar incidence $> 70^\circ$, long shadows.
7. **`SUN-HIGH` (Noon Illumination)**: Solar incidence $< 15^\circ$, minimal shadow contrast.
8. **`AZI-DIFF` (Opposing Azimuths)**: $|\Delta\varphi| > 90^\circ$ (severe shadow inversion).
9. **`LOW-TEX` (Smooth Lunar Maria)**: Mare Tranquillitatis / Oceanus Procellarum basalt plains.
10. **`HIGH-REL` (Rugged Highland Massifs)**: South Pole-Aitken Basin / crater rim walls.
11. **`IIRS-SPEC` (Cross-Spectral Hyperspectral)**: IIRS (250 bands) ↔ LRO WAC panchromatic.

---

# 9. Risk Register & Failure Mitigation

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                            OPERATIONAL RISK REGISTER & CONTINGENCIES                     │
├────┬─────────────────────────────┬──────┬──────┬──────┬──────────────────────────────────┤
│ #  │ Risk Event                  │ Lik. │ Imp. │ Score│ Architectural Mitigation         │
├────┼─────────────────────────────┼──────┼──────┼──────┼──────────────────────────────────┤
│ R1 │ Pretrained LightGlue weights│ MED  │ HIGH │  6   │ Synthetic lunar domain           │
│    │ underperform on regolith    │      │      │      │ fine-tuning via ASP ground truth │
│    │ texture without fine-tuning │      │      │      │ and DEM-rendered shadow pairs.   │
├────┼─────────────────────────────┼──────┼──────┼──────┼──────────────────────────────────┤
│ R2 │ Low keypoint repeatability  │ MED  │ MED  │  4   │ Supplement SuperPoint with       │
│    │ on featureless lunar maria  │      │      │      │ Phase Congruency keypoints.      │
├────┼─────────────────────────────┼──────┼──────┼──────┼──────────────────────────────────┤
│ R3 │ Crater detector fails on    │ LOW  │ HIGH │  3   │ Integrate multi-scale DeepMoon + │
│    │ high-noise polar imagery    │      │      │      │ TMC-2 crater U-Net as fallback.  │
├────┼─────────────────────────────┼──────┼──────┼──────┼──────────────────────────────────┤
│ R4 │ Sub-pixel phase correlation │ MED  │ LOW  │  2   │ Quality gate: if peak < 0.35,    │
│    │ decorrelates on flat patch  │      │      │      │ reject shift and retain coarse.  │
├────┼─────────────────────────────┼──────┼──────┼──────┼──────────────────────────────────┤
│ R5 │ Direct 320× OHRC↔IIRS match │ HIGH │ MED  │  6   │ Transitive chaining via          │
│    │ fails to converge           │      │      │      │ intermediate LRO NAC/WAC bridges.│
├────┼─────────────────────────────┼──────┼──────┼──────┼──────────────────────────────────┤
│ R6 │ GPU hardware acceleration   │ LOW  │ MED  │  2   │ Autonomous fallback to CPU-ready │
│    │ unavailable in environment  │      │      │      │ LNIFT + Phase Correlation engine.│
└────┴─────────────────────────────┴──────┴──────┴──────┴──────────────────────────────────┘
```

---

# 10. Software Integration Layout & File Architecture

```
/Abhi/Projects/SIH/
├── MASTER_SYSTEM_ARCHITECTURE.md     # Master System Design Specification
├── pipeline/
│   ├── __init__.py
│   ├── config.py                     # Hyperparameters, EPSG codes, thresholds
│   ├── layer0_ingestion.py           # ISIS3/ASP wrappers & geodetic BBox cropper
│   ├── layer1_preprocessing.py       # Branches A (Optical), B (Cross-Modal), C (Polar)
│   ├── layer2_router.py              # Terrain classifier & dynamic route selector
│   ├── layer3_matchers/
│   │   ├── __init__.py
│   │   ├── base.py                   # Abstract Matcher Base Class
│   │   ├── primary_lightglue.py      # SuperPoint + PC Feats + LightGlue GNN
│   │   ├── topological_cnsfm.py      # YOLOv9 / DeepMoon + CNSFM Geometric Matcher
│   │   ├── baseline_lnift.py         # High-Speed LNIFT Classical Baseline
│   │   └── transitive_chain.py       # 320x OHRC->NAC->WAC->IIRS Transitive Chaining
│   ├── layer4_spatial_anms.py        # ANMS-SSC Uniformity & 8x8 Entropy Scorer
│   ├── layer5_robust_solver.py       # Data-Seeded DEGENSAC + TPS Mesh + Z-Score Filter
│   ├── layer6_subpixel.py            # Gaussian 2D Paraboloid Fit & Gauss-Helmert Covariance
│   └── layer7_product_eval.py        # GeoTIFF/ISIS Warp & 20% Held-Out Validation Engine
├── tests/
│   ├── test_preprocessing.py         # Unit tests for percentile clipping & resampling
│   ├── test_lightglue_matcher.py     # CUDA/CPU inference tests
│   ├── test_subpixel_refinement.py   # Paraboloid peak fitting verification
│   └── test_full_pipeline.py         # End-to-end integration test
├── scripts/
│   ├── download_pradan_samples.py    # ISSDC PRADAN / QuickMap data puller
│   ├── generate_synthetic_pairs.py   # Controlled affine/illumination test generator
│   └── benchmark_all.py              # Automated test harness for all 11 categories
└── app.py                            # Streamlit Interactive Dashboard
```

---

# 11. Phased 10-Week Implementation Roadmap

```
Week  1–2: Stage 0 Data Ingestion (ISIS3/ASP) + SIFT/RANSAC Baseline + Synthetic Harness
Week  3–4: Stage 3A LightGlue + SuperPoint Integration + Stage 4 ANMS-SSC Implementation
Week  5–6: Stage 3B CNSFM Crater Topology Engine + Polar Route Integration
Week    7: Stage 6 Sub-Pixel Paraboloid Refinement + Gauss-Helmert Covariance
Week    8: Transitive Chaining (OHRC↔IIRS) + Streamlit Dashboard UI
Week 9–10: 10-Config Ablation Study + ASP Cross-Validation + Final Technical Report
```

---

> [!TIP]
> **Summary for Presentation to ISRO Judges:**
> Lead with the working Streamlit demo on real OHRC-NAC pairs, demonstrate the **Three-Path Complementary Failures** (LightGlue handling 90% of scenes, CNSFM taking extreme 147° polar shadow reversals, and Transitive Chaining solving the 320× scale gap), show the **O(n) ANMS spatial uniformity entropy metric**, and prove sub-pixel precision with the **anti-circular 20% held-out checkpoint validation protocol**.
