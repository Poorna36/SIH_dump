# Multi-Scale Moon Surface Matching (MSM) Master System Architecture

## 1. System Formulation & Objective

The Multi-Scale Moon Surface Matching (MSM) engine provides automated, sub-pixel accurate, illumination-invariant image correspondence between Chandrayaan-2 payloads (OHRC, TMC-2, IIRS) and reference products (LRO NAC, WAC, SELENE).

The architecture integrates multi-modal data ingestion, terrain-adaptive preprocessing, dynamic matcher selection (MSM / L1.5), keypoint spatial declustering, and sub-pixel surface refinement.

```
 Chandrayaan-2 Source Products          Reference Orbiter Products
 (OHRC / TMC-2 / IIRS)                 (LRO NAC / WAC Mosaic)
           │                                      │
           └──────────────────┬───────────────────┘
                              ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ L0  DATA & GEOMETRY INGESTION LAYER                         │
 │     ISIS3 / SPICE Kernel Initialization                     │
 │     Footprint Bounding Box Crop & GSD Normalization         │
 └────────────────────────────┬────────────────────────────────┘
                              ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ L1  PREPROCESSING & TERRAIN MASKING LAYER                   │
 │     Shadow/Validity Masking + Wallis Contrast Normalization │
 └────────────────────────────┬────────────────────────────────┘
                              ▼
 ╔═════════════════════════════════════════════════════════════╗
 ║ L1.5 MATCHER SELECTION MODEL (MSM / LightGBM Router)        ║
 ║      13-Feature Extraction Vector x_pair                   ║
 ║      Confidence Routing: Single Matcher vs. Fallback vs. All║
 ╚═════════════════════════════════════════════════════════════╝
                              ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ L2  CORRESPONDENCE ENGINE LAYER                             │
 │     M0: SIFT + Ratio Test       M1: RIFT / RIFT2            │
 │     M2: SuperPoint + LightGlue  M3: CNSFM Crater Branch     │
 └────────────────────────────┬────────────────────────────────┘
                              ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ L3  UNIFORM SPATIAL OPTIMIZATION LAYER                      │
 │     Adaptive Non-Maximum Suppression (ANMS / SSC Variant)   │
 └────────────────────────────┬────────────────────────────────┘
                              ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ L4  GEOMETRIC VERIFICATION & SUB-PIXEL REFINEMENT           │
 │     DEGENSAC Homography Fit + 2D Paraboloid Peak Fitting    │
 └─────────────────────────────────────────────────────────────┘
```

---

## 2. The 15 Core Architectural Principles

| ID | Principle | Technical Formulation & Rationale | Evidence Base |
|---|---|---|---|
| **F1** | **Dual-Stage Spatial Selection** | (a) Keypoint ANMS before matching for sparse matchers; (b) match-level grid selection post-matching. | LoFTR-SPP, SSC |
| **F2** | **Geometric Bounds Verification** | Explicit in-image-domain bounds check $(x \in [0,W], y \in [0,H])$ on predicted correspondences. | LoFTR extrapolation failure |
| **F3** | **Hierarchical Model Ladder** | Fit similarity $\to$ affine $\to$ homography $\to$ tile-wise local transformations sequentially. | SIFT-IIRS-WAC, DESCA |
| **F4** | **Preprocessing Branching Rule** | Learned matchers skip heavy radiometric preprocessing; classical matchers undergo Wallis+CLAHE. | Traditional vs. DL Benchmark |
| **F5** | **Deep-Shadow Masking** | Shadow mask $\sigma_I^2 < \tau_{\text{shadow}}$ computed prior to keypoint extraction; masked pixels excluded. | KAZE, CNSFM |
| **F6** | **Two-Tier Match Set Extraction** | Strict NNDR ($\tau=0.7$) for initialization; loose NNDR ($\tau=1.0$) for consensus evaluation. | DESCA |
| **F7** | **Fourier Phase-Correlation Recipe** | Gaussian apodization windowing, Gaussian pyramid multi-scale, 2D paraboloid sub-pixel peak fitting. | Hybrid Phase Correlation |
| **F8** | **Multi-Metric Quality Vector** | Match count is never a quality proxy; report RMSE, inlier ratio, and spatial coverage std-dev. | Evaluation Protocol |
| **F9** | **Geographic Cell Disjoint Rule** | Benchmark splits enforced over disjoint geographic latitude/longitude tiles ($10^\circ \times 10^\circ$). | ML Hygiene |
| **F10** | **Strict Provenance & PDS Rules** | Maintain ISRO `.img`/`.xml` file names without mutation for ISIS3 SPICE kernel ingestion. | ASP 3.7.0 Pipeline |
| **F11** | **Terrain-Adaptive Routing** | Dynamic fallback (learned $\to$ classical when confidence $< 0.40$; crater branch gated by $\rho_c$). | CNSFM, MSM Router |
| **F12** | **Separate Photometric IIRS Track** | Photometric phase function correction for hyperspectral bands prior to SIFT registration against WAC. | SIFT-IIRS-WAC |
| **F13** | **Spatial Uniformity Metric** | Grid cell population standard deviation $\sigma_{\text{cell}}$ measured alongside RMSE and inlier count. | Problem Statement |
| **F14** | **Gated Crater Branch Execution** | Crater-density check ($\rho_c \ge 5 \text{ craters/tile}$); branch executes only in crater-rich terrain. | CNSFM ($72.3\%$ polar SR) |
| **F15** | **Geo-Disjoint MSM Model Training** | MSM selector trained on geo-cell disjoint splits, activating only after full benchmark validation. | ML Leakage Prevention |

---

## 3. Matcher Selection Model (MSM / L1.5) Formulation

The MSM router evaluates a 13-dimensional feature vector $\mathbf{x} \in \mathbb{R}^{13}$ extracted from image metadata and basic image statistics:

$$\mathbf{x} = \left[ \text{sensor\_pair}, \text{gsd\_ratio}, |\text{lat}|, \Delta\phi_{\text{sun}}, \text{terrain\_class}, \rho_c, f_{\text{shadow}}, f_{\text{overlap}}, C_{\text{src}}, C_{\text{ref}}, G_{\text{src}}, G_{\text{ref}}, N_{\text{tiles}} \right]^\top$$

### LightGBM Multi-Class Classifier
MSM predicts class probabilities $P(M_k | \mathbf{x})$ over 4 matcher branches ($M_0: \text{SIFT}, M_1: \text{RIFT2}, M_2: \text{SuperGlue}, M_3: \text{CNSFM}$):

$$\hat{M} = \arg\max_{k} P(M_k | \mathbf{x})$$

### Execution Routing Policy
- **High Confidence ($P(\hat{M} | \mathbf{x}) \ge 0.65$):** Execute routed matcher $\hat{M}$ only.
- **Moderate Confidence ($0.40 \le P(\hat{M} | \mathbf{x}) < 0.65$):** Execute routed matcher $\hat{M}$ + fallback classical matcher $M_1$.
- **Low Confidence ($P(\hat{M} | \mathbf{x}) < 0.40$):** Execute all matchers in parallel (Benchmark Safe Mode).
- **Hard Gate Constraint:** $M_3$ (CNSFM) blocked if crater density $\rho_c < 5$; $M_2$ (SuperGlue) blocked if GPU unavailable.
