# Technical Architecture & System Documentation
## Multi-Modal, Sun-Angle & Scale-Invariant Lunar Image Correspondence Suite (SIH 26166)

---

## 1. Executive Summary & Hard Problem Formulations

The **Multi-Sensor Lunar Image Registration Software Suite** provides automated, sub-pixel spatial registration for Chandrayaan-2 payloads (**OHRC** ~0.25 m, **TMC-2** ~5 m, **IIRS** ~80 m) against reference basemaps (**LRO NAC/WAC**, **SELENE TC/MI**, **LOLA DTM**).

### The Four Primary Engineering Challenges & Solutions

| # | Challenge | Root Cause | Architectural Solution | Mathematical Basis |
|---|---|---|---|---|
| 1 | **Sun-Angle Variation** | Moving shadows alter pixel intensity gradients | **Phase Congruency & CNSFM Geometry** | Local phase alignment in frequency domain is invariant to amplitude/illumination changes |
| 2 | **Scale Mismatch ($>300\times$)** | Resolution ratio OHRC (0.25m) vs IIRS (80m) | **Chained Multi-Sensor Bridge (OHRC $\rightarrow$ TMC-2 $\rightarrow$ IIRS)** + **MS-RIFT** | Multi-octave Log-Gabor scale-space prevents resolution-gap aliasing |
| 3 | **Multi-Modal Differences** | Cross-spectral optical vs hyperspectral/SAR | **Maximum Index Map (MIM) & LNIFT** | Orientation index of maximum Log-Gabor response yields structural maps |
| 4 | **Planar Degeneracy & Relief** | Flat lunar maria cause standard RANSAC to fail | **Data-Seeded DEGENSAC + TPS Mesh** | Data-driven seeding avoids DE divergence; DEGENSAC detects degenerate coplanar subsets |

---

## 2. Complete System Pipeline Architecture

```text
+--------------------------------------------------------------------------+
|                 LAYER 0 — GEODETIC DATA INFRASTRUCTURE                   |
|   USGS ISIS3 (spiceinit, isisimport) -> NASA ASP (orthorectification)    |
|   SPICE Ephemeris -> Expanded Geodetic BBox (S = 2.5 * sigma_ephemeris)   |
|   NASA Trek WMTS API / Local Raster Cache Auto-Cropping                  |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|                LAYER 1 — PREPROCESSING & RESAMPLING                      |
|  Bicubic / Bilinear adaptive GSD Resampling to Common Resolution        |
|  Branch A (Panchromatic): CLAHE (Clip = 2.0, Grid = 8x8) + Gaussian     |
|  Branch B (Cross-Modal): Log-Gabor Filter Bank -> Maximum Index Map (MIM)|
|  Branch C (Polar Minimal): 1%-99% Percentile Stretch + Bilateral Filter |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|                  LAYER 2 — ADAPTIVE ROUTING ENGINE                       |
|                                                                          |
|  Compute: Crater Density (Dc), Texture Entropy (Htext), Solar Azimuth (dphi)|
|                                                                          |
|  IF Dc > Tau_crater AND dphi > 90deg  --> PATH A (CNSFM Geometry)        |
|  IF Htext > Tau_entropy AND dphi <= 60deg --> PATH B (MS-RIFT + PhaseCorr) |
|  IF Htext <= Tau_entropy OR Polar     --> PATH C (LightGlue / RoMa v2)   |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|               LAYER 3 — MULTI-PATH CORRESPONDENCE ENGINE                 |
|                                                                          |
|  PATH A: CNSFM (Crater Neighborhood Signature Matching)                  |
|    YOLOv9 Crater Center & Diameter Detection -> KNN Topological Graphs   |
|    Similarity-Invariant Angle & Distance Ratios -> MCR Filter            |
|                                                                          |
|  PATH B: MS-RIFT (Multi-Scale Phase Congruency) + Phase Correlation      |
|    4-Octave Log-Gabor Scale Space -> Maximum Index Map (MIM) Descriptor  |
|    Multi-Scale Gaussian Pyramid Phase Correlation + NNDR (Lowe = 0.75)  |
|                                                                          |
|  PATH C: Learned Transformer Matching (LightGlue / RoMa v2)              |
|    SuperPoint / DISK Front-End -> GNN Dual-Softmax Optimal Transport    |
|    Mandatory Spatial Bounds Check: 0 <= x < width AND 0 <= y < height   |
|                                                                          |
|  ENSEMBLE ROUTER: Parallel SIFT baseline logged; Match-level voting     |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|           LAYER 4 — SPATIAL UNIFORMITY ENFORCEMENT                       |
|  ANMS via Suppress-via-Size-Clustered (SSC) Algorithm                    |
|  Enforce Point Budget (N_max = 500) + 8x8 Tile Grid Coverage Verification|
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|             LAYER 5 — TWO-LAYER SOLVER & OUTLIER REJECTION               |
|  Stage 1: Learned Pre-Filter (LightGlue Dustbin p > 0.2 / MCR Filter)    |
|  Stage 2: Data-Seeded DESCA-DEGENSAC Geometric Verification              |
|           - Seeded from top-10 confidence matches (no random DE)        |
|           - Flat Mare -> Affine (6 DOF) + DEGENSAC                       |
|           - Oblique -> Homography (8 DOF) + DEGENSAC                     |
|           - Rugged Highland -> Piecewise Affine Thin-Plate Spline (TPS)  |
|  Stage 3: GCP Declustering (min 15px spacing) + Residual Z-Score Filter  |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|             LAYER 6 — MULTI-SCALE SUB-PIXEL REFINEMENT                   |
|  Gaussian-Windowed Phase Correlation on 31x31 px Window                  |
|  2D Paraboloid Surface Fitting over 3x3 Correlation Peak                 |
|  Gauss-Helmert Covariance Propagation -> Formal Standard Errors (sx, sy) |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|         LAYER 7 — GEOMETRIC TRANSFORM, PRODUCT & EVALUATION              |
|  GDAL Warp to Reference CRS (Selenographic Equirectangular / Polar Stere) |
|  Exports: GeoTIFF + ISIS .cub + Tie-Point CSV + Execution Metadata JSON  |
|  20% Held-Out Checkpoint Validation (Anti-Circularity Protocol)          |
|  Metrics: RMSE_x, RMSE_y, RMSE_total, Inlier Ratio, Spatial Entropy Index|
+--------------------------------------------------------------------------+
```

---

## 3. Tool Matrix & External Software Integrations

The system integrates flight-proven planetary processing software, deep learning frameworks, and geospatial tools:

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                      EXTERNAL TOOL INTEGRATION MATRIX                    │
├─────────────────┬─────────────────────────────┬──────────────────────────┤
│ Tool / Software │ Primary Purpose             │ Execution Interface     │
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ USGS ISIS3      │ SPICE Ephemeris / Geodetic  │ CLI Subprocesses        │
│                 │ Camera Calibration          │ (isisimport, spiceinit)  │
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ NASA ASP        │ Orthorectification &        │ CLI Subprocesses        │
│                 │ Bundle Adjustment           │ (mapproject, bundle_adj) │
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ GDAL / Rasterio │ BBox Query, Cropping, GSD   │ Python Bindings & C API │
│                 │ Resampling, GeoTIFF Warping │ (gdal.Warp, rasterio)   │
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ NASA Moon Trek  │ Automated Reference Tile    │ REST / WMTS HTTP API    │
│ WMTS API        │ Retrieval by Lat/Lon BBox   │ (requests / urllib3)     │
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ PyTorch         │ Deep Matcher Inference      │ Python API              │
│                 │ (SuperPoint, LightGlue, RoMa)│ (torch.cuda streams)     │
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ OpenCV (cv2)    │ CLAHE, Gaussian Smoothing,  │ Python API              │
│                 │ SIFT Baseline, RANSAC       │ (cv2.createCLAHE)        │
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ PyDEGENSAC      │ Degeneracy-Aware RANSAC for │ Python C-Extension      │
│                 │ Flat Lunar Maria            │ (pydegensac.findHomography)│
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ Scikit-Image    │ Phase Correlation, 2D Peak  │ Python API              │
│                 │ Fitting, Spatial Entropy    │ (skimage.registration)  │
├─────────────────┼─────────────────────────────┼──────────────────────────┤
│ YOLOv9          │ Deep Crater Detection for   │ PyTorch Model Weights   │
│                 │ CNSFM Topological Matching  │ (ultralytics.YOLO)      │
└─────────────────┴─────────────────────────────┴──────────────────────────┘
```

### Detailed Tool Functionalities

1. **USGS ISIS3 (Integrated Software for Imagers and Spectrometers)**:
   * **`isisimport`**: Converts raw PDS3/PDS4 data files from Chandrayaan-2 ISSDC archives into ISIS `.cub` cube files.
   * **`spiceinit`**: Computes exact SPICE kernel positioning (spacecraft trajectory, camera orientation, target body geometry).
   * **`cam2map`**: Projects native camera sensor coordinates into standard lunar map projections (Selenographic Equirectangular or Polar Stereographic).

2. **NASA Ames Stereo Pipeline (ASP)**:
   * **`mapproject`**: Orthorectifies raw source image strips onto reference DTMs (e.g., LOLA or SELENE DTMs).
   * **`bundle_adjust`**: Performs rigorous geodetic tie-point bundle adjustment across multiple overlapping image strips, anchoring final pixel coordinates to the LOLA control network.

3. **GDAL / Rasterio**:
   * Performs metadata-driven bounding box queries, virtual raster tiling (VRT), bicubic GSD resampling, and final GeoTIFF warping with nodata masking.

4. **NASA Moon Trek WMTS API (`trek.nasa.gov`)**:
   * Ingests the padded geodetic bounding box $[\text{Lat}_{\min}, \text{Lat}_{\max}, \text{Lon}_{\min}, \text{Lon}_{\max}]$ and queries the global LROC WAC/NAC tiles over HTTP, retrieving target reference crops automatically without manual file downloads.

5. **PyTorch + LightGlue / RoMa v2**:
   * CUDA-accelerated feature extraction (SuperPoint) and graph neural network matching (LightGlue).
   * Runs in non-blocking PyTorch CUDA streams for ultra-low latency ($< 150\text{ ms}$ per patch).

6. **PyDEGENSAC**:
   * Solves degenerate coplanar configuration problems on flat lunar maria, preventing standard RANSAC from over-fitting to trivial planar match subsets.

---

## 4. Process Working Lifecycle & Concurrency Architecture

### Complete Operational Lifecycle

```text
  [ Raw Source Data ]       [ Reference Basemap ]
           │                          │
           ▼                          ▼
  ┌────────────────────────────────────────────────┐
  │ STEP 1: Ingestion & Ephemeris Footprint Calc   │
  │ Read SPICE metadata -> Expand BBox by 2.5x     │
  └───────────────────────┬────────────────────────┘
                          │
                          ▼
  ┌────────────────────────────────────────────────┐
  │ STEP 2: Automatic Reference Patch Crop & Resamp│
  │ Query NASA Moon Trek WMTS -> Resample GSD      │
  └───────────────────────┬────────────────────────┘
                          │
                          ▼
  ┌────────────────────────────────────────────────┐
  │ STEP 3: Sensor-Pair Classification & Routing   │
  │ Compute Dc, Htext, dphi -> Route to Path A/B/C │
  └───────────────────────┬────────────────────────┘
                          │
                          ▼
  ┌────────────────────────────────────────────────┐
  │ STEP 4: Parallel Matcher Engine Execution      │
  │ Thread 1 (GPU): Primary Matcher (LightGlue/MIM)│
  │ Thread 2 (CPU): Baseline SIFT Logger           │
  └───────────────────────┬────────────────────────┘
                          │
                          ▼
  ┌────────────────────────────────────────────────┐
  │ STEP 5: Spatial Uniformity Filter (ANMS SSC)   │
  │ Enforce 500 point budget + 8x8 tile coverage   │
  └───────────────────────┬────────────────────────┘
                          │
                          ▼
  ┌────────────────────────────────────────────────┐
  │ STEP 6: Two-Layer DEGENSAC Geometric Solver    │
  │ Seed from top-10 matches -> Fit Affine/Homog/TPS│
  └───────────────────────┬────────────────────────┘
                          │
                          ▼
  ┌────────────────────────────────────────────────┐
  │ STEP 7: Sub-Pixel Paraboloid Peak Refinement   │
  │ Phase Correlation -> 3x3 Paraboloid Fit -> sx,sy│
  └───────────────────────┬────────────────────────┘
                          │
                          ▼
  ┌────────────────────────────────────────────────┐
  │ STEP 8: Product Warp & Anti-Circularity Eval   │
  │ GDAL GeoTIFF Warp -> 20% Held-Out Validation   │
  └────────────────────────────────────────────────┘
```

### Concurrency & Threading Model

To ensure real-time performance and prevent CPU/GPU bottlenecks, execution is parallelized across asynchronous worker threads:

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                      ASYNC PROCESS CONCURRENCY MODEL                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│ Main Orchestrator                                                        │
│   │                                                                      │
│   ├──► CUDA Stream 0 (GPU Worker):                                       │
│   │      SuperPoint Feature Extraction + LightGlue Graph Inference      │
│   │                                                                      │
│   ├──► CPU Worker Pool Thread 1:                                         │
│   │      Log-Gabor Filter Bank & Maximum Index Map (MIM) Generation     │
│   │                                                                      │
│   ├──► CPU Worker Pool Thread 2:                                         │
│   │      Parallel SIFT Baseline Execution & Logging                     │
│   │                                                                      │
│   └──► Thread Barrier Synchronization                                    │
│          Combine Match Sets -> Pass to ANMS SSC Filter & DEGENSAC Solver │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Mathematical Specifications by Layer

### LAYER 0 — Geodetic ROI Ingestion & Footprint Expansion
Reads sensor ephemeris metadata or invokes ISIS3 `spiceinit`. To guarantee spatial overlap despite ephemeris pointing uncertainty $\sigma_{\text{ephemeris}}$, the bounding box is expanded by:
$$\Delta \text{Deg} = \frac{k \cdot \sigma_{\text{ephemeris}}}{R_{\text{Moon}} \cdot \frac{\pi}{180^\circ}}, \quad (k=2.5, R_{\text{Moon}} \approx 1737.4\text{ km})$$

---

### LAYER 1 — Preprocessing & Multi-Scale Log-Gabor Filter Bank
To achieve radiation and illumination invariance, images are transformed into Phase Congruency space using a multi-scale, multi-orientation Log-Gabor filter bank:
$$G_{o,s}(\omega, \theta) = \exp\left( -\frac{(\ln(\omega/\omega_o))^2}{2(\ln(\sigma_{\omega}/\omega_o))^2} \right) \cdot \exp\left( -\frac{(\theta - \theta_{o,s})^2}{2\sigma_{\theta}^2} \right)$$
Phase Congruency $PC(x,y)$ is calculated via local phase alignment:
$$PC(x,y) = \frac{\sum_o \sum_s W_o(x,y) \lfloor A_{o,s}(x,y) \Delta \Phi_{o,s}(x,y) - T_o \rfloor_+}{\sum_o \sum_s A_{o,s}(x,y) + \epsilon}$$
The **Maximum Index Map (MIM)** descriptor selects the orientation index $o^*$ maximizing amplitude response:
$$\text{MIM}(x,y) = \arg\max_{o} \left( \sum_s A_{o,s}(x,y) \right)$$

---

### LAYER 2 — Adaptive Classifier & Routing Rules

```text
                                Extract Metadata:
               Crater Density (Dc), Texture Entropy (Htext), Solar Azimuth (dphi)
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           ▼                            ▼                            ▼
  Dc > Tau_crater AND          Htext > Tau_entropy AND          Htext <= Tau_entropy OR
     dphi > 90deg                    dphi <= 60deg                   Polar Shadowing
           │                            │                            │
           ▼                            ▼                            ▼
        PATH A                       PATH B                       PATH C
   (CNSFM Geometry)           (MS-RIFT + PhaseCorr)       (LightGlue / RoMa v2)
```

---

### LAYER 3 — Correspondence Matcher Suite

#### Path A: CNSFM (Crater Neighborhood Signature Matching)
1. Detect craters via YOLOv9: Centers $C_i = (x_i, y_i)$, Radii $r_i$.
2. Form KNN topological triples $(C_0, C_i, C_j)$ and compute similarity-invariant features:
   $$\alpha = \angle C_i C_0 C_j, \quad \rho = \frac{\|C_0 - C_i\|}{\|C_0 - C_j\|}, \quad \gamma = \frac{r_i}{r_j}$$
3. Mismatched CNSF Removal (MCR) verifies local similarity transformation consensus.

#### Path B: MS-RIFT (Multi-Scale Log-Gabor RIFT)
1. Computes MIM descriptors across a 4-octave Log-Gabor scale space.
2. Performs coarse-to-fine Gaussian pyramid Phase Correlation matching.
3. Applies Nearest Neighbor Distance Ratio (NNDR Lowe ratio = 0.75).

#### Path C: LightGlue / RoMa v2 (Learned Transformer)
1. SuperPoint extracts keypoints $\mathbf{p}_i$ and 256D descriptors $\mathbf{d}_i$.
2. LightGlue GNN solves dual-softmax optimal transport with adaptive early stopping.
3. **Mandatory Bounds Validation**: Discard matches where $x < 0 \lor x \ge W \lor y < 0 \lor y \ge H$.

---

### LAYER 4 — Spatial Uniformity Enforcement (ANMS SSC)
For $N$ detected matches sorted by confidence score $c_i$, find suppressing radius $r_i$:
$$r_i = \min_{j: c_j > c_i} \| \mathbf{p}_i - \mathbf{p}_j \|$$
Binary search over radius $r$ selects top $N_{\text{max}} = 500$ spatially uniform keypoints. Enforces $\ge 80\%$ tile coverage across an $8 \times 8$ grid.

---

### LAYER 5 — Two-Layer Outlier Rejection & DEGENSAC Solver

1. **Stage 1 (Learned Pre-Filter)**: LightGlue dustbin probability threshold ($p > 0.2$) or MCR structural check.
2. **Stage 2 (Data-Seeded DEGENSAC)**:
   * Seeded from top-10 confidence matches to prevent Differential Evolution (DE) divergence.
   * Flat Mare $\rightarrow$ Affine 6-DOF + DEGENSAC.
   * Oblique / Moderate Relief $\rightarrow$ Homography 8-DOF + DEGENSAC.
   * Rugged Highland $\rightarrow$ Piecewise Affine Thin-Plate Spline (TPS) Mesh:
     $$f(x,y) = a_1 + a_x x + a_y y + \sum_{i=1}^K w_i \left( \|\mathbf{p} - \mathbf{p}_i\|^2 \ln \|\mathbf{p} - \mathbf{p}_i\| \right)$$

---

### LAYER 6 — Multi-Scale Sub-Pixel Peak Fitting & Gauss-Helmert Covariance

1. Window $W_s, W_r$ by 2D Hanning function $H(x,y)$ to prevent edge ringing.
2. Compute Cross-Power Spectrum $R(u,v) = \frac{\mathcal{F}\{W'_s\} \mathcal{F}^*\{W'_r\}}{|\mathcal{F}\{W'_s\} \mathcal{F}^*\{W'_r\}|}$ and inverse transform $r(x,y) = \mathcal{F}^{-1}\{R(u,v)\}$.
3. **2D Paraboloid Peak Fitting**: Fit surface $P(x,y) = A x^2 + B y^2 + C x + D y + E x y + F$ over $3 \times 3$ peak neighborhood:
   $$\delta x = \frac{2 B C - E D}{E^2 - 4 A B}, \quad \delta y = \frac{2 A D - E C}{E^2 - 4 A B}$$
4. **Gauss-Helmert Covariance Propagation**: Compute covariance matrix $\mathbf{\Sigma}_{p}$ yielding formal standard errors $\sigma_x, \sigma_y$.

---

### LAYER 7 — Product Generation & Held-Out Evaluation

#### Anti-Circularity Protocol
Randomly partition verified tie-points into **80% Model Fitting** and **20% Held-Out Validation Checkpoints**. All evaluation metrics are calculated exclusively on the held-out checkpoints:

$$\text{CheckPoint RMSE}_{\text{total}} = \sqrt{ \frac{1}{N_{\text{val}}} \sum_{i=1}^{N_{\text{val}}} \left( (\hat{x}_i - x_i)^2 + (\hat{y}_i - y_i)^2 \right) }$$

$$\text{Spatial Shannon Entropy } H = -\sum_{k=1}^M p_k \log_2(p_k)$$

---

## 6. Software Layout & File Mapping

```text
/home/dark/Downloads/IDP/antigravity/SIH 2026/
├── ARCHITECTURE_DOCUMENTATION.md
├── pipeline/
│   ├── __init__.py
│   ├── config.py                 # Hyperparameters, GSD ratios, CRS defaults
│   ├── layer0_infrastructure.py  # ISIS3/ASP data import & padded ROI cropping
│   ├── layer1_preprocessing.py   # Preprocessing Branches A (Optical), B (Cross-Modal), C (Polar)
│   ├── layer2_routing.py         # Crater density + entropy terrain classifier
│   ├── layer3_matchers/
│   │   ├── __init__.py
│   │   ├── base.py               # Abstract Base Class for Matchers
│   │   ├── path_a_cnsfm.py       # Geometry/Crater Neighborhood Structure Matcher
│   │   ├── path_b_ms_rift.py     # Multi-Scale Log-Gabor RIFT + Phase Correlation
│   │   ├── path_c_learned.py     # LightGlue / RoMa v2 Matcher Wrapper
│   │   └── ensemble.py           # Match-level voting & baseline logger
│   ├── layer4_uniformity.py      # ANMS (SSC) & 8x8 Grid Verification
│   ├── layer5_outliers.py        # Data-Seeded DESCA + DEGENSAC Solver (Affine/Homography/TPS)
│   ├── layer6_refinement.py      # Sub-Pixel Paraboloid Fitting + Gauss-Helmert Covariance
│   └── layer7_output.py          # GeoTIFF/ISIS Warp & 20% Held-Out Evaluation Engine
├── tests/                        # PyTest Suite for Layers 0-7
└── run_pipeline.py               # CLI Entry Point
```
