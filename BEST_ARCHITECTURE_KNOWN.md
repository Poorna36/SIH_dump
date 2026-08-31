# SIH26166 — MASTER SYSTEM ARCHITECTURE & MATCHING ENGINE (v2.0 Complete)

**Multi-modal, Sun-angle & scale-invariant image correspondence using Chandrayaan-2 optical images
(OHRC / TMC-2 / IIRS) vs LRO (NAC / WAC) reference**

> **Document Scope:** This single, consolidated master document contains the complete, authoritative specification for the SIH26166 registration pipeline, including the core multi-matcher engine, data ingest, spatial selection, model ladder, sub-pixel refinement, evaluation harness, and the lightweight **Matcher Selection Model (MSM / L1.5)**.

---

## 0. The 15 Architectural Fixes

Every architectural design decision traces to empirical evidence from the 15 research files in `SIH_dump` and live data checks.

| # | Fix | Description & Rationale | Source Evidence |
|---|---|---|---|
| F1 | **Dual-Stage Spatial Selection** | (a) Keypoint ANMS (SSC variant) *before* matching for sparse matchers (detect → ANMS → describe → match); (b) match-level grid/coverage selection *after* matching for all matchers. | Supplementary research §1; LoFTR-SPP |
| F2 | **Mandatory Geometric Sanity Check** | In-image-domain bounds + one-to-one constraint enforced on every learned match set. | HybridPhaseCorrelation: LoFTR produces out-of-domain extrapolated matches |
| F3 | **Hierarchical Model Ladder** | Fit similarity → affine → homography → tile-wise local models in sequence. | SIFT-IIRS-WAC, DESCA, HybridPhaseCorrelation |
| F4 | **No Preprocessing Overspend** | Learned matchers skip heavy preprocessing (CLAHE/shadow normalization); matcher must carry illumination robustness. | Traditional-vs-DL: classical methods failed polar despite preprocessing |
| F5 | **Deep-Shadow Validity Masking** | Shadow/low-information mask computed from solar geometry + local variance before matching; masked pixels excluded. | KAZE (PC limit in textureless shadow); CNSFM |
| F6 | **DESCA Two-Tier Match Set** | Strict $NNDR \approx 0.7$ for initialization + loose $NNDR \approx 1.0$ as evaluation pool. | DESCA takeaway #5 |
| F7 | **Phase-Correlation Recipe** | Gaussian/Tukey apodization windows (never Blackman), Gaussian pyramid coarse-to-fine, 2D paraboloid peak fit. | HybridPhaseCorrelation takeaway #7 |
| F8 | **Multi-Metric Quality Proxy** | Match count is never a proxy for quality. RMSE, inlier ratio, and spatial coverage reported together on held-out points. | Traditional-vs-DL takeaway #6 |
| F9 | **Geographic Leakage Rule** | Benchmark splits by disjoint geographic cells ($10^\circ \times 10^\circ$ lon/lat), never random pair splits. | Standard ML hygiene |
| F10 | **Strict Provenance Rules** | Never rename ISRO `.img`/`.xml` products (breaks `isisimport`); use ASP $\ge 3.7.0$ conda env with per-orbit-date CK kernels. | ASP §8.15 Chandrayaan-2 example |
| F11 | **Terrain-Adaptive Arbitration** | Confidence-based fallback (learned $\to$ classical when learned confidence low; crater branch gated). | CNSFM; SuperGlue hybrid hypothesis |
| F12 | **Separate IIRS Track** | Photometric correction (incidence/emission/phase) $\to$ SIFT-class registration vs WAC; sub-80m target. | Supplementary research §7 |
| F13 | **Spatial Coverage First-Class Metric** | Grid density std-dev / percent cells populated measured alongside RMSE and inlier ratio. | Problem statement requirement; LoFTR-SPP |
| F14 | **Gated Crater Branch** | Crater-density check first ($\ge \tau_c$); branch activates only in crater-rich terrain. | CNSFM: 72.3% SR at south pole vs 31.2% baseline |
| F15 | **Geo-Cell Disjoint MSM Training** | MSM training uses geo-cell-disjoint splits (same F9 rule) and activates only after full benchmark validation. | Standard ML hygiene; prevents selector leakage |

---

## 1. Master System Architecture Diagram

```
 Chandrayaan-2 source products            Reference products
 (PRADAN/CHMAP zips: .img+.xml,           LROC NAC EDR/CDR strips,
  IIRS QUB — never renamed)               WAC 643nm global mosaic (ISIS cube)
        │                                        │
        └───────────────┬────────────────────────┘
                        ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ L0  DATA & GEOMETRY LAYER                                     │
 │  isisimport → spiceinit (ASP 3.7.0 env, per-date CK kernels)  │
 │  label parser → footprint (corner lat/lon), solar incidence/  │
 │  azimuth, UTC, GSD                                            │
 │  padded bbox (2–5× pointing uncertainty)                      │
 │  reference query: Lunar ODE bbox search / Moon Trek WMTS /    │
 │  GDAL crop of local WAC mosaic                               │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼  Pair = source patch + reference patch + PairRecord
 ┌───────────────────────────────────────────────────────────────┐
 │ L1  PREPROCESSING                                             │
 │  shadow/validity mask (F5) → radiometric normalization        │
 │  (percentile clip + stat transfer) → sensor branch:           │
 │    OHRC→NAC: CLAHE        TMC-2→WAC: histogram-match + CLAHE  │
 │    (learned matchers skip heavy branches — F4)                │
 │  tiling for scene heterogeneity; GSD reconciliation pyramid   │
 │  → writes L1 meta.json (masked_fraction, patch stats, tiles)  │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼  PairRecord + L1 meta.json
 ╔═══════════════════════════════════════════════════════════════╗
 ║ L1.5  MATCHER SELECTION MODEL  (MSM)                          ║
 ║  FeatureVector ← {sensor_pair, gsd_ratio, latitude_abs,       ║
 ║    delta_solar_azimuth, terrain_class, crater_density,        ║
 ║    masked_fraction, overlap_fraction, texture_contrast×2,     ║
 ║    mean_gradient×2, tile_count}  (13 features)                ║
 ║  Model: LightGBM multi-class classifier (4 classes = matchers)║
 ║  → SelectorResult {selected_matcher, confidence, fallback,    ║
 ║                    all_probs, routing_reason}                  ║
 ║  Routing:                                                      ║
 ║    confidence ≥ τ_high (0.65) → run selected_matcher only     ║
 ║    confidence ∈ [τ_low, τ_high) → run fallback_matcher        ║
 ║    confidence < τ_low (0.40)  → run all matchers (safe mode)  ║
 ║  Hard rules: M3 blocked if crater_density < τ_c;              ║
 ║              M2 blocked if no GPU                              ║
 ║  Benchmark mode: selector bypassed; all matchers run          ║
 ╚══════════════════════════════════════════════════════════════╝
                            ▼  routing decision
 ┌───────────────────────────────────────────────────────────────┐
 │ L2  CORRESPONDENCE ENGINE  (pluggable, benchmarked)           │
 │  M0 SIFT + ratio test          (baseline, always runs in      │
 │                                 benchmark mode; fallback)      │
 │  M1 RIFT/RIFT2 (PC + MIM)      (classical illumination-robust)│
 │  M2 SuperPoint + LightGlue     (learned, GPU)                 │
 │  M3 Crater-geometry branch     (gated by crater density, F14) │
 │  Interface: detect/describe/match → kpts, matches, scores     │
 │  In MSM production mode: only the routed matcher runs         │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ L3  UNIFORM CORRESPONDENCE OPTIMIZATION                       │
 │  (a) keypoint ANMS (SSC) pre-match for M0/M1        (F1a)     │
 │  (b) confidence filter → image grid → per-cell max-N →       │
 │      coverage-aware greedy selection                (F1b)     │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ L4  GEOMETRIC VERIFICATION & MODEL ESTIMATION                 │
 │  DEGENSAC/MAGSAC → model ladder: similarity → affine →        │
 │  homography → tile-wise local models                 (F3)     │
 │  in-domain bounds + one-to-one constraint            (F2)     │
 │  optional: DESCA DE-refinement with two-tier sets    (F6)     │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ L5  SUB-PIXEL REFINEMENT                                      │
 │  local NCC / phase correlation around each match; Gaussian/   │
 │  Tukey window; multi-scale pyramid; 2D paraboloid peak (F7)   │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ L6  PRODUCT GENERATION                                        │
 │  warp source → registered GeoTIFF; match-points CSV/GCP;      │
 │  checkerboard + overlay QC images                             │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ L7  EVALUATION HARNESS  (decides the winning matcher)         │
 │  RMSE (held-out), pct<1px, pct<0.5px, MedAE, inlier          │
 │  count/ratio, spatial coverage, runtime; precision/recall     │
 │  → leaderboard → MSM training labels (train split only)       │
 │  → MSM benchmark: selector accuracy, RMSE degradation,        │
 │    runtime reduction, fallback rate            (F15)          │
 └──────────────────────────┬────────────────────────────────────┘

 Side track (parallel, own frontend — F12):
   IIRS QUB → photometric correction → SIFT-class match vs WAC
   → sub-80 m RMSE (sub-pixel at 80 m GSD)
```

---

## 2. Layer Specifications

### L0 — Data & Geometry Layer
- **Inputs**: PRADAN/CHMAP zips (`.img` + `.xml`, PDS4; IIRS = `QUB`). Original filenames preserved (F10).
- **Processing**:
  1. Unzip to `data/raw/` (never rename files).
  2. `isisimport` $\to$ `.cub`; `spiceinit` / CSM via ASP-bundled ISIS + ALE + USGSCSM. Fetch CK kernels per orbit date only.
  3. XML Label parser $\to$ per-product metadata: corner lat/lon footprint, solar incidence & azimuth, UTC timestamp, GSD, product ID.
  4. Padded bounding box = footprint expanded $2\text{--}5\times$ pointing uncertainty ($k \cdot \sigma_{\text{pointing}}$).
  5. Reference acquisition: Lunar ODE bounding box search for LROC NAC strips; GDAL crop of local WAC 643 nm mosaic.
- **Output**: `PairRecord` written to `data/pairs/manifest.jsonl`.

### L1 — Preprocessing
- **Shadow/validity mask (F5)**: Evaluated from solar incidence/azimuth + local intensity variance; excludes textureless cast shadows.
- **Radiometric normalization**: Percentile clipping ($2\%/98\%$) $\to$ min-max $\to$ mean/std transfer toward reference patch.
- **Sensor branches**: OHRC $\to$ NAC uses CLAHE + PCA; TMC-2 $\to$ WAC uses histogram matching + CLAHE. Learned matchers M2/M3 skip heavy branches (F4).
- **Tiling & Reconciliation**: Overlapping tiles handle local terrain heterogeneity; pyramid resampling reconciles GSD differences.
- **L1 `meta.json`**: Writes pre-computed image statistics consumed by MSM (`masked_fraction`, `src_texture_contrast`, `ref_texture_contrast`, `src_mean_gradient`, `ref_mean_gradient`, `tile_count`).

### L1.5 — Matcher Selection Model (MSM)
Sits between L1 and L2 to select the optimal single matcher before execution.
- **Benchmark Mode (`msm.enabled: false`)**: Transparent pass-through. All matchers execute; leaderboard data is collected.
- **Production Mode (`msm.enabled: true`)**: Assembles `FeatureVector` from `PairRecord` + L1 `meta.json`. Runs LightGBM inference ($< 1$ ms CPU). Emits `SelectorResult` and routes execution.
- **Routing Thresholds**:
  - Confidence $\ge \tau_{\text{high}} (0.65) \implies$ Execute `selected_matcher` only (`high_confidence`).
  - Confidence $\in [\tau_{\text{low}}, \tau_{\text{high}}) (0.40 \le c < 0.65) \implies$ Execute `fallback_matcher` (2nd highest probability).
  - Confidence $< \tau_{\text{low}} (0.40) \implies$ Execute all matchers (`very_low_confidence_full_mode`).
- **Hard Rule Gating**: M3 probability forced to 0 if `crater_density` $< \tau_c$ ($5.0$ craters/Mpx); M2 probability forced to 0 if GPU is unavailable. Proabilities renormalize before threshold evaluation.

### L2 — Correspondence Engine
All matchers implement the unified `Matcher` abstract interface:
- **M0 SIFT + Ratio Test**: Tiled SIFT $\to$ Lowe ratio 0.75 $\to$ RANSAC homography. Always runs in benchmark mode and serves as safe fallback.
- **M1 RIFT / RIFT2**: Phase congruency detection + Maximum Index Map (MIM) descriptor + multi-octave log-Gabor scale-space extension.
- **M2 SuperPoint + LightGlue**: Pretrained deep feature extractor + graph neural network matcher. F2 sanity checks mandatory.
- **M3 Crater-Geometry Branch**: Detector (YOLOv9 transfer / DeepMoon) $\to$ Crater Neighborhood Structure Map (CNSF) $\to$ similarity-invariant geometric matching $\to$ MCR structural outlier removal. Gated by crater density (F14).

### L3 — Uniform Correspondence Optimization
1. **Keypoint ANMS (F1a)**: SSC variant (Suppression via Square Covering) applied pre-match for sparse matchers M0/M1.
2. **Grid Partition & Coverage Selection (F1b)**: Divide image into $N \times N$ cells $\to$ cap max-$N$ matches per cell $\to$LoFTR-SPP style greedy bisection selection to hit match budget $K$ while maximizing populated cell fraction.
3. **Outputs**: Filtered match set, `spatial_coverage` score, and `grid_density_std`.

### L4 — Geometric Verification & Model Estimation
- **DEGENSAC / MAGSAC++**: Handles planar degenerate cases in flat maria. Reprojection threshold $t_{\text{gsd}}$ scales with reference GSD.
- **Model Ladder (F3)**: Sequentially fits similarity (4-DoF) $\to$ affine (6-DoF) $\to$ homography (8-DoF). Accepts simplest model meeting RMSE criteria; falls back to tile-wise local models on high-relief or $> \pm 55^\circ$ polar terrain.
- **F2 Checks**: Enforces in-domain coordinate bounds and strict 1-to-1 match constraints.
- **GCP Declustering**: Min-spacing grid filtering and Z-score residual filtering.

### L5 — Sub-Pixel Refinement
- Local patch extraction ($32 \times 32$ px) around coarse matches.
- Normalized cross-correlation (NCC) or Phase-Only Correlation (POC) with Tukey/Gaussian apodization windows (F7).
- Integer peak $\to$ 2D paraboloid surface fit $\to$ sub-pixel displacement offset $(\Delta x, \Delta y)$. Peak sharpness threshold $\tau_q$ filters ambiguous multi-modal peaks.

### L6 — Product Generation
- Warps source image onto reference grid using fitted transform $\to$ **Registered GeoTIFF**.
- Exports **Match-Points Product**: CSV + GCP list (pixel coordinates in both images + lon/lat georeference).
- Generates Quality Control (QC) artifacts: checkerboard overlay, residual heatmap, side-by-side match overlay.

### L7 — Evaluation Harness
- Leaderboard generation per pair $\times$ matcher $\times$ stratum (latitude $\times$ $\Delta$-azimuth $\times$ terrain).
- **Metrics**: RMSE (held-out checkpoints), $\% < 1\text{px}$, $\% < 0.5\text{px}$, MedAE, inlier count, inlier ratio, spatial coverage, runtime.
- **MSM Evaluation (`--mode msm_eval`)**: Computes selector accuracy, top-2 accuracy, mean RMSE degradation vs oracle best, runtime reduction, and fallback rate on held-out test split.

---

## 3. Matcher Selection Model (MSM) Features & Contracts

### 3.1 Feature Vector Specification (13 Features)

| # | Feature Name | Type | Range / Values | Extraction Source & Transformation |
|---|---|---|---|---|
| 1 | `sensor_pair_enc` | int | 0=OHRC-NAC, 1=TMC-WAC, 2=IIRS-WAC | Categorical encoding from PairRecord |
| 2 | `gsd_ratio` | float | $(0.0, 1.0]$ | `src_gsd / ref_gsd` from PairRecord |
| 3 | `latitude_abs` | float | $[0.0^\circ, 90.0^\circ]$ | Absolute latitude of footprint centroid |
| 4 | `delta_solar_azimuth` | float | $[0.0^\circ, 180.0^\circ]$ | $|\text{src\_azimuth} - \text{ref\_azimuth}|$, clamped |
| 5 | `terrain_class_enc` | int | 0=highland, 1=maria, 2=polar, 3=mixed | Categorical encoding from PairRecord |
| 6 | `crater_density` | float | $[0, \infty)$ craters/Mpx | $\log(1 + \text{crater\_density})$ transformed |
| 7 | `masked_fraction` | float | $[0.0, 1.0]$ | Fraction of shadow/invalid pixels in L1 mask |
| 8 | `overlap_fraction` | float | $(0.0, 1.0]$ | Overlap fraction between source and reference |
| 9 | `src_texture_contrast` | float | $[0, \infty)$ DN | Mean local $\sigma$ in $8\times 8$ windows (source) |
| 10 | `ref_texture_contrast` | float | $[0, \infty)$ DN | Mean local $\sigma$ in $8\times 8$ windows (reference) |
| 11 | `src_mean_gradient` | float | $[0, \infty)$ DN/px | Mean Sobel gradient magnitude (source) |
| 12 | `ref_mean_gradient` | float | $[0, \infty)$ DN/px | Mean Sobel gradient magnitude (reference) |
| 13 | `tile_count` | int | $[1, \sim 100]$ | Count of valid tiles post-reconciliation |

### 3.2 Data Contracts & Structures (Python Data Classes)

```python
# Feature Vector Contract
@dataclass
class FeatureVector:
    sensor_pair_enc:      int
    gsd_ratio:            float
    latitude_abs:         float
    delta_solar_azimuth:  float
    terrain_class_enc:    int
    crater_density:       float   # log1p transformed
    masked_fraction:      float
    overlap_fraction:     float
    src_texture_contrast: float
    ref_texture_contrast: float
    src_mean_gradient:    float
    ref_mean_gradient:    float
    tile_count:           int

    def to_array(self) -> np.ndarray:
        return np.array([getattr(self, f) for f in FEATURE_NAMES], dtype=np.float32)

# Selector Result Contract (saved to results/<pair_id>/selector.json)
@dataclass
class SelectorResult:
    pair_id:              str
    selected_matcher:     str         # "sift" | "rift2" | "lightglue" | "crater"
    confidence:           float       # max(all_probs)
    fallback_matcher:     str         # 2nd highest probability matcher
    all_probs:            dict        # {"sift": p0, "rift2": p1, "lightglue": p2, "crater": p3}
    routing_reason:       str         # "high_confidence" | "low_confidence_fallback" | ...
    matchers_to_run:      list[str]   # list of matchers passed to L2 engine
    hard_rules_applied:   list[str]   # e.g., ["crater_density_gate"]
    selector_version:     str         # "msm_v1"
    feature_vector_hash:  str         # SHA256 of feature vector bytes
```

---

## 4. Configuration Reference

### 4.1 `configs/msm.yaml` Schema

```yaml
msm:
  enabled: false                      # Production mode toggle (false until validated)
  model_path: "models/msm_v1.pkl"     # LightGBM model pipeline path
  model_stats_path: "models/msm_v1_stats.json"
  model_version: "msm_v1"

  # Confidence Thresholds
  tau_high: 0.65                      # Single matcher execution threshold
  tau_low: 0.40                       # Fallback matcher execution threshold

  # Hard Rule Overrides
  hard_rules:
    crater_density_gate:
      enabled: true
      tau_c: 5.0                      # Craters / Mpx threshold
    gpu_gate:
      enabled: true
      check_at_startup: true
    iirs_track_gate:
      enabled: true

  # Fallback Strategy
  fallback:
    on_model_load_error: "benchmark_mode"
    on_feature_extraction_error: "benchmark_mode"
    on_s4_gate_failure: "sift"         # Safe fallback if routed matcher fails S4
    log_all_fallback_events: true

  # Training Hyperparameters
  training:
    min_labeled_pairs: 50
    composite_score_weights:
      rmse_norm_inv: 0.50
      inlier_ratio: 0.25
      spatial_coverage: 0.25
    lightgbm_params:
      objective: "multiclass"
      num_class: 4
      metric: "multi_logloss"
      n_estimators: 200
      learning_rate: 0.05
      max_depth: 6
      num_leaves: 31
      random_state: 42
```

### 4.2 Additions to `configs/matchers.yaml`

```yaml
arbitration:
  order: [crater, lightglue, rift2, sift]
  fallback_floor: {inlier_ratio: 0.05, min_inliers: 20}
  msm_safe_fallback: "sift"
  msm_matcher_order: [sift, rift2, lightglue, crater]
```

---

## 5. Executable Pipeline Runbook & CLI Interface

### 5.1 Pipeline Stages (S0 to S9)

```
 S0 Environment Setup      conda activate asp; export ISISDATA=...
       │
       ▼
 S1 Ingestion (L0)         python scripts/ingest.py --raw data/raw --out data/calibrated
       │
       ▼
 S2 Pair Building (L0)     python scripts/build_pairs.py --products data/metadata/products.jsonl
       │
       ▼
 S3 Preprocessing (L1)     python scripts/preprocess.py --manifest data/pairs/manifest.jsonl
       │
       ▼
 S4.5 MSM Routing (L1.5)   python -m src.selector.model --pair <pair_id>
       │
       ▼
 S4 Matching (L2)          python scripts/benchmark.py --pair <pair_id> [--use-msm]
       │
       ▼
 S5 Selection (L3)         src/selection/spatial.py (ANMS + coverage grid)
       │
       ▼
 S6 Verification (L4)      src/registration/ladder.py (DEGENSAC + model ladder)
       │
       ▼
 S7 Sub-Pixel Refine (L5)  src/refinement/local.py (NCC/POC + paraboloid peak)
       │
       ▼
 S8 Products (L6)          python scripts/register.py --pair <pair_id> --matcher <matcher>
       │
       ▼
 S9 Evaluation (L7)        python -m src.evaluation.aggregate --out results/leaderboard.csv
```

### 5.2 Command Line Interface Quick Reference

```bash
# 1. Full benchmark run across all matchers (MSM disabled - data collection):
python scripts/benchmark.py --config configs/ohrc_nac.yaml --splits test --matchers sift,rift2,lightglue,crater

# 2. Train Matcher Selection Model (requires >= 50 labeled pairs):
python scripts/train_msm.py --leaderboard results/leaderboard.csv --manifest data/pairs/manifest.jsonl --split msm_split --split-value train --out models/msm_v1.pkl

# 3. Perform geographic leakage audit:
python -m src.evaluation.leakage_audit --manifest data/pairs/manifest.jsonl --check-msm

# 4. Run MSM evaluation benchmark on test split:
python scripts/benchmark.py --config configs/ohrc_nac.yaml --splits test --mode msm_eval --selector models/msm_v1.pkl --out results/msm_benchmark.csv

# 5. Execute production pipeline with MSM routing enabled:
python scripts/benchmark.py --config configs/ohrc_nac.yaml --use-msm --selector models/msm_v1.pkl
```

---

## 6. Validation Protocol & Acceptance Criteria

Before `msm.enabled` can be set to `true`, the trained MSM must pass **all 8 Acceptance Criteria (AC1–AC8)** on the held-out test split:

| # | Metric | Acceptance Threshold | Purpose / Rationale |
|---|---|---|---|
| AC1 | **Selector Accuracy** | $\ge 70.0\%$ | Percentage of test pairs where MSM selects the true oracle-best matcher. |
| AC2 | **Top-2 Accuracy** | $\ge 85.0\%$ | Percentage where true best matcher is in `[selected_matcher, fallback_matcher]`. |
| AC3 | **Mean RMSE Degradation** | $\le +0.10\text{ px}$ | Mean increase in RMSE over oracle-best matcher across test pairs. |
| AC4 | **Max RMSE Degradation** | $\le +0.50\text{ px}$ | Prevents catastrophic misrouting on individual test pairs. |
| AC5 | **Runtime Reduction** | $\ge 50.0\%$ | Mean processing time savings compared to all-matchers baseline. |
| AC6 | **Fallback Rate** | $\le 20.0\%$ | Percentage of pairs triggering `very_low_confidence_full_mode`. |
| AC7 | **Feature Importance** | Non-trivial | Top-5 features must exhibit non-zero LightGBM gain. |
| AC8 | **Leakage Audit** | Exit Code 0 | Zero shared $10^\circ \times 10^\circ$ geo-cells between MSM train and test splits. |

---

## 7. Architecture Decision Records (ADRs)

- **DEC-001**: *Pluggable Matcher Engine Architecture.* All matchers implement `Matcher` and pass through identical L3–L7 evaluation to enable fair benchmarking.
- **DEC-002**: *Dual-Stage Spatial Selection.* Keypoint ANMS (SSC) + post-match grid coverage selection to enforce uniform spatial distribution.
- **DEC-003**: *Padded Bounding Box Search.* Coarse SPICE footprint expanded $2\text{--}5\times$ pointing uncertainty to crop reference patch before matching.
- **DEC-004**: *L1.5 MSM Position.* Sits between L1 Preprocessing and L2 Engine to eliminate running all matchers in production.
- **DEC-005**: *LightGBM Classifier.* Selected for multi-class tabular classification due to native categorical support and $<1$ ms CPU latency.
- **DEC-006**: *Dual Threshold & Fallback Policy.* High/low confidence thresholds ($\tau_{\text{high}}=0.65, \tau_{\text{low}}=0.40$) with escalation to M0 SIFT on gate failures.
- **DEC-007**: *Strict Geo-Cell Disjointness (F15).* MSM training labels restricted strictly to train-split geographic tiles ($10^\circ \times 10^\circ$).
- **DEC-008**: *Operational Activation Gate.* `msm.enabled` remains `false` until all 8 Acceptance Criteria pass on held-out test data.

---

## 8. Implementation Roadmap (Phases P0 to P7)

```
 P0 Infrastructure Setup    ASP 3.7.0 conda env, ISISDATA, configs
       │
       ▼
 P1 Dataset Ingestion       PRADAN/CHMAP download, label parser, build_pairs.py
       │
       ▼
 P2 Baseline Pipeline       M0 SIFT end-to-end (L0–L7), GT tie-points on pilot set
       │
       ▼
 P3 Dataset Scaling         Expand to 50 pairs across strata, GT on 15-20 pairs
       │
       ▼
 P4 Matcher Integration     M1 RIFT2, M2 SuperPoint+LightGlue, M3 Crater branch
       │
       ▼
 P5 Evaluation & Products   Leaderboard generation, GeoTIFF warping, arbitration policy
       │
       ▼
 P5.5 MSM Phase             Feature extraction, LightGBM training, msm_eval, activation
       │
       ▼
 P6 IIRS Track             Photometric correction -> SIFT vs WAC (sub-80m target)
       │
       ▼
 P7 User Interface          Web App UI (only after backend pipeline is validated)
```

---

## 9. Deliverables & Artifact Mapping

| SIH Problem Statement Requirement | Satisfying Component | Evidence & Validation Artifact |
|---|---|---|
| Multi-modal correspondence (CH-2 vs LRO/SELENE) | L2 Engine (M1 RIFT2, M2 LightGlue, M3 Crater) | `results/leaderboard.csv` per sensor pair |
| Sub-pixel registration accuracy | L5 Sub-Pixel Refinement (NCC/POC + paraboloid) | `rmse_px`, `pct_lt_1px`, `pct_lt_0p5px` columns |
| Uniform spatial distribution of match points | L3 ANMS (SSC) + Coverage Grid Selection | `spatial_coverage` & `grid_density_std` columns |
| High throughput / Operational efficiency | L1.5 Matcher Selection Model (MSM) | `results/msm_benchmark.csv` (runtime reduction $\ge 50\%$) |
| Match-Points Deliverable Product | L6 Product Generation | `match_points.csv` and `match_points.gcp` |
| Registered Image Deliverable Product | L6 Product Generation | `registered.tif` GeoTIFF & QC overlays |
| Geometric Outlier Rejection | L4 DEGENSAC + Model Ladder + GCP Declustering | `inlier_ratio` and `geometry.json` |
