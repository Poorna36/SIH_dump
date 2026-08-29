# SIH 26166 — System Architecture
## Multi-Modal, Sun-Angle & Scale-Invariant Lunar Image Correspondence

---

## 0. Problem Statement (Distilled)

Given images from Chandrayaan-2 payloads (**OHRC** ~0.25 m, **TMC-2** ~5 m, **IIRS** ~80 m) and reference data from **LRO** (NAC ~0.5 m, WAC ~100 m) / **SELENE**, produce:

1. **Reliable corresponding match points** across sensor pairs
2. With **sub-pixel accuracy** (RMSE < 1 px)
3. **Uniformly distributed** across the image
4. Robust to **Sun-angle/illumination variation**, **scale differences** (up to ~300x), **viewpoint changes**, and **multi-modal radiometric differences**
5. With quantitative evaluation: **RMSE, inlier count, inlier ratio, spatial distribution metric**

### The Four Hard Sub-Problems (Ranked by Evidence of Difficulty)

| # | Challenge | Severity | Evidence |
|---|-----------|----------|----------|
| 1 | **Sun-angle / illumination variation** | Critical — dominant error source | SIFT fails at poles (0% SR); even "illumination-robust" methods (HAPCG, ML-HLMO) degrade severely; shadow-driven overlap loss is qualitatively distinct from brightness shift |
| 2 | **Scale mismatch** (up to ~300x OHRC-IIRS) | High | No paper tests extreme scale ratios; RIFT explicitly lacks scale invariance; upscaling introduces artifacts |
| 3 | **Multi-modal / cross-sensor radiometric differences** | High | Optical-SAR fails for all classical methods; OHRC-IIRS is cross-spectral, not just cross-resolution |
| 4 | **Terrain relief / geometric distortion** | Medium-High | Global affine models fail on mountains (DESCA); RMSE grows near-exponentially beyond +/-55 deg latitude |

---

## 1. Architecture Overview — Hybrid Adaptive Pipeline

```
+--------------------------------------------------------------------------+
|                        LAYER 0 — DATA INFRASTRUCTURE                     |
|         ISIS3 (spiceinit, calibration) -> ASP (import, ortho)            |
|         Metadata-driven reference patch auto-selection                   |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|                    LAYER 1 — PREPROCESSING & NORMALIZATION                |
|  Georeferencing -> Resolution resampling -> 8-bit normalization          |
|  Radiometric normalization (percentile clip + stat-transfer)             |
|  Sensor-adaptive enhancement (CLAHE for optical / log-transform IIRS)   |
|  Optional: cGAN illumination front-end (if training data available)      |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|                      LAYER 2 — TERRAIN-ADAPTIVE ROUTING                  |
|                                                                          |
|  Classify scene into one of three regimes using crater density +         |
|  texture entropy + Sun-angle metadata:                                   |
|                                                                          |
|  +-----------------+  +-----------------+  +----------------------+      |
|  |  CRATER-RICH    |  |  TEXTURED       |  |  LOW-TEXTURE /       |      |
|  |  (highlands,    |  |  (mixed terrain,|  |  EXTREME ILLUM.      |      |
|  |  older surfaces)|  |  equatorial)    |  |  (mare, polar shadow)|      |
|  +--------+--------+  +--------+--------+  +-----------+----------+      |
|           |                    |                        |                 |
|           v                    v                        v                 |
|     +----------+       +------------+          +------------+            |
|     |  PATH A  |       |   PATH B   |          |   PATH C   |            |
|     | Geometry |       |  Hybrid    |          |   Learned  |            |
|     |  -based  |       | Classical  |          | Detector-  |            |
|     |          |       |            |          |   free     |            |
|     +----------+       +------------+          +------------+            |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|                   LAYER 3 — CORRESPONDENCE ENGINE(S)                     |
|                                                                          |
|  PATH A: CNSFM-style                                                     |
|    YOLOv9 crater detection -> Crater Neighborhood Structure (CNSF)       |
|    -> Similarity-invariant geometric matching -> MCR outlier removal     |
|    Strength: Mathematically illumination-invariant (zero dependence on   |
|              pixel intensity). 72.3% SR at south pole vs 17% SIFT.       |
|    Weakness: Fails in crater-sparse terrain.                             |
|                                                                          |
|  PATH B: Hybrid Classical (primary workhorse)                            |
|    +--------------------------------------------------------------+      |
|    |  TILING: Divide into overlapping tiles (adaptive size)       |      |
|    |  ----------------------------------------------------------- |      |
|    |  DETECTION: Fusion multi-detector                            |      |
|    |    - Phase-congruency-weighted keypoints (I-KAZE style)      |      |
|    |    - + FAST (dense) + Shi-Tomasi (stable)                    |      |
|    |    - Filtered by crater-rim/ridge proximity                  |      |
|    |      (replacing Hough-line filter for lunar terrain)         |      |
|    |  ----------------------------------------------------------- |      |
|    |  DESCRIPTION: MIM descriptor (RIFT/RIFT2 style)              |      |
|    |    - Log-Gabor filter bank -> Maximum Index Map              |      |
|    |    - Illumination-invariant by construction                  |      |
|    |  ----------------------------------------------------------- |      |
|    |  MATCHING: Phase correlation (multi-scale Gaussian           |      |
|    |    pyramid, Gaussian/Tukey apodization)                      |      |
|    |    + NNDR (Lowe ratio 0.75) for descriptor matches           |      |
|    +--------------------------------------------------------------+      |
|    Strength: No training data needed; proven on satellite imagery;       |
|              competitive with LoFTR on VHR data.                         |
|    Weakness: Struggles at poles; no cross-spectral evidence.             |
|                                                                          |
|  PATH C: Learned / Detector-free                                         |
|    +--------------------------------------------------------------+      |
|    |  OPTION C1 — LightGlue (SuperGlue successor)                |      |
|    |    SuperPoint/DISK front-end -> GNN attention-based          |      |
|    |    matching -> Sinkhorn optimal transport + dustbin           |      |
|    |    outlier rejection. Adaptive depth (early stopping).       |      |
|    |                                                              |      |
|    |  OPTION C2 — LoFTR / RoMa                                   |      |
|    |    Detector-free, coarse-to-fine transformer matching.       |      |
|    |    Dense/semi-dense correspondences. Sub-pixel via fine      |      |
|    |    feature maps. Needs bounds-validation post-step.          |      |
|    |                                                              |      |
|    |  OPTION C3 — MatchAnything (stretch goal)                    |      |
|    |    Large-scale multi-domain pretrained. Explicitly           |      |
|    |    targets cross-modality generalization. Test off-the-      |      |
|    |    shelf on lunar data first.                                |      |
|    +--------------------------------------------------------------+      |
|    Strength: Only family that succeeds on ALL sensor pairs incl.         |
|              polar + SAR-optical (SuperGlue: 100% coverage).             |
|    Weakness: Needs GPU; domain-specific fine-tuning likely needed;       |
|              can produce out-of-bounds extrapolated matches.             |
|                                                                          |
|  ENSEMBLE FUSION: When multiple paths produce results, fuse via          |
|    match-level voting (keep matches agreed by >=2 paths) or              |
|    confidence-weighted selection (highest match confidence wins).         |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|               LAYER 4 — SPATIAL UNIFORMITY ENFORCEMENT                   |
|                                                                          |
|  Adaptive Non-Maximal Suppression (ANMS / SSC variant)                   |
|    - Suppress keypoints if a stronger one exists within radius r         |
|    - r adapts per-point for even spatial coverage                        |
|    - Fallback: grid-based cap (top-k per NxN cell)                      |
|  ----------------------------------------------------------------       |
|  OR: LoFTR-style greedy NMS merging (bisection-searched radius           |
|      to hit target point budget) + conflict resolution                   |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|                  LAYER 5 — OUTLIER REJECTION (TWO-LAYER)                 |
|                                                                          |
|  Stage 1: Learned/structural pre-filter                                  |
|    - LightGlue dustbin (confidence < 0.2 -> reject)                     |
|    - OR: MCR (Mismatched CNSF Removal) for crater-path matches          |
|    - OR: DESCA (DE-optimization consensus, RMSE 0.67-0.95 px)           |
|                                                                          |
|  Stage 2: Geometric verification                                         |
|    - DEGENSAC (degeneracy-aware RANSAC) — mandatory for flat             |
|      lunar terrain (mare, crater floors) where standard RANSAC           |
|      degenerates on dominant planes                                      |
|    - Per-tile similarity/affine -> global polynomial                     |
|    - Parameters: 0.5 px reprojection threshold, 5000-10000 iter         |
|                                                                          |
|  Stage 3: Post-RANSAC statistical filter                                 |
|    - GCP declustering (min 15-20 px spacing, grid-based,                |
|      keep nearest to cell center)                                        |
|    - Residual Z-score outlier removal (requires >20 GCPs)               |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|                 LAYER 6 — SUB-PIXEL REFINEMENT                           |
|                                                                          |
|  Multi-scale phase correlation (coarse-to-fine Gaussian pyramid)         |
|    -> Per-level 1D parabolic fitting (accumulate across scales)          |
|    -> Final 2D paraboloid fitting on 3x3 correlation peak window         |
|                                                                          |
|  Alternative: MI + DE/best/1/bin fine registration                       |
|    - Affine params seeded from coarse estimate (NOT random)              |
|    - DE mutation/crossover maximizes MI, cubic B-spline histogram        |
|    - Multi-resolution coarse->fine search                                |
|                                                                          |
|  Alternative: cv2.cornerSubPix() or skimage phase_cross_correlation      |
|    (upsampled-DFT / Guizar-Sicairos method)                             |
|                                                                          |
|  KEY PRINCIPLE: Detector + refinement must be co-tuned (demonstrated     |
|  that the same refinement method produces order-of-magnitude different   |
|  accuracy depending on which detector it is paired with).                |
+-------------------------------+------------------------------------------+
                                |
+-------------------------------v------------------------------------------+
|             LAYER 7 — GEOMETRIC TRANSFORMATION & OUTPUT                  |
|                                                                          |
|  Per-tile: similarity/affine transform (for local verification)          |
|  Global: polynomial warp fit by least squares on filtered GCPs           |
|    - Works well < +/-55 deg latitude                                     |
|    - For polar / high-relief: piecewise/local transformation             |
|      or rational function model + DTM-based correction                   |
|  CRS reconciliation (post-warp georeferencing to reference CRS)          |
|  Output: registered image product + correspondence table +               |
|          uncertainty estimates (Gauss-Helmert covariance, optional)       |
+--------------------------------------------------------------------------+
```

---

## 2. Sensor-Pair Strategy Matrix

Not all sensor pairs are the same problem. The architecture routes differently based on the pair:

| Source | Reference | Scale Ratio | Primary Challenge | Recommended Path | Confidence |
|--------|-----------|-------------|-------------------|------------------|------------|
| **OHRC** (0.25 m) | **LRO NAC** (0.5 m) | ~2x | Illumination (same modality) | Path B (hybrid classical) + Path C fallback at poles | High |
| **OHRC** (0.25 m) | **TMC-2** (5 m) | ~17x | Scale + illumination | Path B with multi-scale extension + ANMS | Medium |
| **TMC-2** (5 m) | **LRO WAC** (100 m) | ~20x | Scale + illumination | Path B (tiled SIFT baseline proven here) | Medium-High |
| **IIRS** (80 m) | **LRO WAC** (100 m) | ~1.25x | Cross-spectral (different bands) | Path C (LightGlue/MatchAnything) — separate module | Medium |
| **OHRC** (0.25 m) | **IIRS** (80 m) | ~300x | Extreme scale + cross-spectral | **Out of scope for direct matching** — chain via TMC-2 as intermediary | Low |
| **Any** | **Any** (polar) | Any | Extreme shadow/azimuth diff | Path A (CNSFM) primary + Path C (LightGlue) ensemble | Medium |

> **Key Architectural Decision:** IIRS should be treated as a **separate module** (Phase 2) with its own preprocessing (photometric correction) and matching pipeline, NOT folded into the OHRC/TMC pipeline. The spectral + resolution gap is a qualitatively different sub-problem.

---

## 3. Why This Architecture (Design Rationale)

### 3.1 Why Hybrid / Multi-Path, Not a Single Method?

Every paper reviewed arrives at the same conclusion from different angles:

| Paper | Finding |
|-------|---------|
| Traditional vs DL (OHRC/IIRS benchmark) | SuperGlue is the only method that works on ALL conditions — but needs GPU + fine-tuning |
| CNSFM (crater matching) | Best illumination robustness (72.3% at pole) — but fails in crater-sparse terrain |
| Hybrid Phase Correlation (2026) | Classical pipeline matches LoFTR on VHR satellite data — but same-sensor only |
| RIFT | Best cross-modal descriptor — but explicitly lacks scale invariance |
| DESCA | Best outlier rejection (0.67 px RMSE) — but only optical-optical, affine model fails on terrain relief |

**No single method covers all four challenges simultaneously.** A terrain-adaptive router that selects the best-suited path per scene is the evidence-backed design.

### 3.2 Why Phase Congruency as the Core Detection Principle?

Three independent sources confirm:

1. **RIFT** — PC detection is illumination/contrast-invariant by construction (100% SR vs SIFT's 31.7%)
2. **I-KAZE** — PC-weighting improves keypoint repeatability under cross-sensor differences (beat SURF/FAST/BRISK/ORB/RIFT on all datasets)
3. **CNSFM** — even stronger: bypassing pixel intensity entirely via geometry achieves 72.3% polar SR

Phase congruency operates in the **frequency domain on local phase alignment**, not pixel amplitude — it is theoretically immune to the monotonic/nonlinear intensity shifts caused by Sun-angle changes. This is not just "more robust" than gradient-based methods; it is architecturally incapable of the failure mode that kills SIFT.

### 3.3 Why LightGlue Over SuperGlue?

- Same architecture (GNN + attention + optimal transport), same lab
- **Adaptive depth** — early stopping for easy pairs (~16.9 ms vs SuperGlue's fixed cost)
- Supports **multiple front-ends** (SuperPoint, DISK, ALIKED, SIFT)
- **Apache-2.0 licensed**, actively maintained, integrated into HuggingFace
- **Strictly dominates** SuperGlue on speed with equivalent accuracy

### 3.4 Why DEGENSAC, Not Standard RANSAC?

Lunar terrain contains large **near-planar regions** (mare, crater floors, flat highland plateaus). Standard RANSAC is known to degenerate on dominant planes — it finds the dominant plane's transform and rejects everything else, even if the dominant plane's matches are trivially easy and the interesting/hard matches are off-plane. DEGENSAC explicitly guards against this.

### 3.5 Why Two-Layer Outlier Rejection?

Evidence from SuperGlue paper:
> "Matches are so clean that a non-robust least-squares solver (DLT) outperforms RANSAC" on synthetic homographies

But on real pose tasks, the paper **still uses RANSAC after the dustbin**. The learned filter reduces but does not eliminate the need for geometric verification. Two layers is the defensible design.

---

## 4. Critical Implementation Details

### 4.1 Initialization Sensitivity (DESCA Lesson)

> **Random DE initialization causes divergence and zero correct matches.** Data-driven initialization (via a strict/clean match subset + leave-one-out RMSE pruning) is required.

If using any optimization-based outlier rejection (DE, PSO, etc.), always seed from a high-confidence initial match subset. Never initialize randomly.

### 4.2 Patch/Window Size Trade-off

| Patch Size | Match Density | RMSE | Sub-pixel % |
|------------|---------------|------|-------------|
| 16x16 | High (7271 inliers) | 1.198-1.232 px | Lower |
| 64x64 | Lower | 1.103 px | 56% < 1 px |

**Design choice:** Use **adaptive patch sizes** — larger patches (64x64) in low-texture regions for better accuracy, smaller patches (16-32) in high-texture regions for density. Terrain-adaptive, not fixed.

### 4.3 LoFTR Bounds Validation (Mandatory)

> [REPORTED] "Some correspondences fall outside the image domain" — LoFTR can extrapolate matches beyond valid spatial limits.

**Any LoFTR/RoMa-based component MUST include a bounds check:**

```python
if (match.x < 0 or match.x >= img_width or
    match.y < 0 or match.y >= img_height):
    discard(match)
```

This is a **mandatory safeguard**, not optional.

### 4.4 Sun-Angle Metadata as a Routing Signal

Use solar incidence angle and azimuth difference between image pairs to **route processing**:

| Condition | Route |
|-----------|-------|
| Azimuth diff < 30 deg | Standard Path B (classical hybrid) |
| Azimuth diff 30-90 deg | Path B + Path C ensemble |
| Azimuth diff > 90 deg | Path A (crater geometry) primary; Path C (learned) secondary |
| Incidence angle > 70 deg (grazing) | Flag as degraded; expect lower accuracy ceiling |

### 4.5 Interpolation Method Selection

> **Bicubic is NOT always better than bilinear on lunar imagery.**

| Sun Angle | Recommended Interpolation |
|-----------|--------------------------|
| High (> 45 deg) | Bicubic (finer detail preservation) |
| Low (< 45 deg) | Bilinear (avoids shadow-boundary amplification) |

Select interpolation method conditionally based on illumination geometry metadata, not fixed a priori.

---

## 5. Software Stack

```
+-----------------------------------------------------+
|  DATA LAYER                                          |
|  - USGS ISIS3 (calibration, SPICE, map projection)  |
|  - NASA ASP (import, orthorectification, bundle adj) |
|  - GDAL/rasterio (reference mosaic cropping)         |
|  - Moon Trek WMTS API (reference tile retrieval)     |
+-----------------------+-----------------------------+
                        |
+-----------------------v-----------------------------+
|  ALGORITHM LAYER (Python)                            |
|  - OpenCV (SIFT, FAST, Shi-Tomasi, CLAHE, RANSAC)   |
|  - scikit-image (phase_cross_correlation, metrics)   |
|  - PyTorch (LightGlue, LoFTR/RoMa inference)        |
|  - YOLOv9 (crater detection — transfer-learned)      |
|  - ANMS-Codes (spatial uniformity enforcement)       |
|  - pydegensac (DEGENSAC implementation)              |
|  - Custom: MIM descriptor, CNSF matching, routing    |
+-----------------------+-----------------------------+
                        |
+-----------------------v-----------------------------+
|  EVALUATION LAYER                                    |
|  - RMSE (X, Y, total) — on held-out checkpoints     |
|  - Inlier count & inlier ratio                       |
|  - Sub-pixel accuracy % (< 1 px and < 0.5 px)       |
|  - MedAE (median absolute error)                     |
|  - Spatial distribution: std-dev of density per grid |
|  - Runtime per image pair                             |
|  - Optional: Gauss-Helmert covariance/uncertainty     |
+-----------------------------------------------------+
```

---

## 6. Evaluation Strategy

### 6.1 Ground Truth

| Method | Use Case | Notes |
|--------|----------|-------|
| **Synthetic transforms** | Week 1 pipeline testing | Apply known affine/homography to real image; exact ground truth |
| **Held-out checkpoints** | Primary evaluation | Reserve 20% of matches; fit transform on 80%; measure error on 20% |
| **LRO NAC cross-check** | Final validation | Compare registered output against independently georeferenced NAC DEM |
| **ASP bundle adjustment residuals** | Secondary | Already-proven OHRC stereo pairs provide accuracy benchmarks |

### 6.2 Terrain-Stratified Reporting

Report accuracy **separately** for terrain classes:

| Terrain Class | Lunar Analog | Expected RMSE Range |
|---------------|-------------|---------------------|
| Gentle (< 15 deg slope) | Mare, smooth plains | < 0.5 px |
| Moderate (15-30 deg) | Moderate highlands | 0.5-1.5 px |
| Rugged (> 30 deg) | Crater rims, mountains | 1.5-3.0 px (geometry-limited) |
| Polar shadow | Permanently shadowed regions | 2.0-5.0+ px (information-limited) |

> **Key insight from literature:** In rugged terrain, *"the registration error is primarily governed by the imaging geometry rather than by the registration algorithm itself."* Report this honestly — it is a physics limit, not a pipeline failure.

---

## 7. Phased Development Plan

### Phase 1 — Core Pipeline (Week 1)
- [ ] Set up ISIS3 + ASP environment; ingest sample OHRC + TMC-2 data
- [ ] Implement Path B (hybrid classical): tiled PC-weighted detection -> MIM descriptor -> phase correlation matching -> RANSAC -> sub-pixel refinement
- [ ] Implement ANMS for spatial uniformity
- [ ] Implement evaluation suite (RMSE, inlier ratio, spatial std-dev)
- [ ] Validate on synthetic transforms first, then real OHRC-NAC equatorial pairs

### Phase 2 — Robustness Extension (Week 2)
- [ ] Integrate LightGlue (Path C) as learned matcher
- [ ] Implement terrain-adaptive routing (crater density + texture entropy classifier)
- [ ] Implement CNSFM-style crater matching (Path A) for polar scenes
- [ ] Implement DEGENSAC + two-layer outlier rejection
- [ ] Test on polar OHRC-NAC pairs
- [ ] Ensemble fusion logic (match-level voting across paths)

### Phase 3 — IIRS Module & Polish (if time permits)
- [ ] IIRS photometric correction preprocessing
- [ ] IIRS-WAC registration (separate pipeline, SIFT-based baseline)
- [ ] Final terrain-stratified evaluation report
- [ ] Uncertainty estimation (Gauss-Helmert)

---

## 8. What NOT to Do (Evidence-Backed Anti-Patterns)

| Anti-Pattern | Evidence | Alternative |
|--------------|----------|-------------|
| Use SIFT alone as primary matcher | Failed at poles in 3 independent papers (0-33% SR) | PC-based detection + MIM descriptor, or LightGlue |
| Rely on preprocessing to rescue gradient-based descriptors | Heavy CLAHE/inversion/dilation still -> 0% SR at poles | Use inherently illumination-invariant methods |
| Naively fuse SIFT + ORB descriptors (IntFeat approach) | Underperforms plain SIFT; amplifies noise | If hybrid, fuse at match/decision level, not descriptor level |
| Use single global affine transform | Fails on terrain relief (mountains, craters) | Piecewise/per-tile transforms + polynomial global warp |
| Use SSIM/PSNR as primary registration metric | Doesn't measure geometric accuracy (only visual similarity) | RMSE + inlier ratio + spatial distribution |
| Use histogram/CLAHE alone for illumination robustness | Cannot model nonlinear Sun-angle-driven illumination changes | Phase congruency or learned features |
| Initialize DE/optimization randomly | Demonstrated divergence to zero matches | Always seed from high-confidence match subset |
| Trust LoFTR output without bounds checking | Can produce out-of-image-domain matches | Mandatory spatial validity check |
| Use Blackman apodization for phase correlation | Suppresses too much high-frequency information | Gaussian or Tukey windows |
| Treat IIRS as same pipeline as OHRC/TMC | Cross-spectral is a qualitatively different problem | Separate module with photometric correction |
| Report a single global accuracy number | Hides terrain-dependent accuracy degradation | Terrain-stratified reporting |
| Post-hoc outlier classifiers (PointCN/OANet style) | Cannot recover matches NN search missed; capped by NN recall | Joint context-aware matching (LightGlue/LoFTR) |

---

## 9. Key Architectural Differentiators for SIH Judges

1. **Terrain-adaptive routing** — no other reviewed system selects correspondence method based on scene characteristics. This is a genuine, evidence-backed architectural contribution.

2. **Three-path ensemble** — geometry (CNSFM) + classical hybrid (PC+MIM+phase correlation) + learned (LightGlue) covers the full challenge space. Each path has a demonstrated failure mode that the other two compensate for.

3. **Phase congruency as unifying principle** — used in detection (I-KAZE), description (MIM/RIFT), and matching (phase correlation). Theoretically illumination-invariant at every stage, not just one.

4. **Two-layer outlier rejection** (learned/structural pre-filter + DEGENSAC) — more robust than either alone; explicitly handles lunar flat-terrain degeneracy.

5. **Honest, terrain-stratified evaluation** — reporting accuracy per terrain class with physics-driven accuracy ceilings is more scientifically credible than a single number.

6. **Built on flight-proven infrastructure** (ISIS3 + ASP) — dramatically reduces implementation risk and strengthens feasibility argument. Novel contribution is the matching layer, not the data pipeline.

---

## 10. Open Research Questions

1. **Scale-invariant RIFT/MIM** — extending RIFT with multi-octave log-Gabor scale-space is a genuine open contribution; no paper demonstrates this.
2. **TMC-2 as intermediary** — chaining OHRC -> TMC-2 -> IIRS via TMC-2 as a bridge sensor is architecturally sound but untested.
3. **Lunar-domain LightGlue fine-tuning** — generating synthetic training data (known transforms on real lunar images, or rendered imagery from DEMs with controlled Sun angles) is likely necessary but unvalidated.
4. **Crater detection transfer** — YOLOv9 crater detector is trained on LROC NAC at 0.5 m; retraining for OHRC (0.25 m) and TMC-2 (5 m) is needed.
5. **MatchAnything on lunar data** — completely untested; could be the most direct solution to cross-modal matching if it generalizes.
