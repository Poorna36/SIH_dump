# SIH26166 — MSM FULL ARCHITECTURE & SYSTEM SPECIFICATION (Master Single File)

**Multi-modal, Sun-angle & scale-invariant image correspondence using Chandrayaan-2 optical images (OHRC / TMC-2 / IIRS) vs LRO (NAC / WAC) reference**

> **Single-File Master Specification:** This document merges all architectural specs, pipeline runbooks, feature definitions, configuration schemas, interface data contracts, validation/benchmark protocols, implementation plans, and decision records for the Matcher Selection Model (MSM) into one single file.

---

# TABLE OF CONTENTS
1. [Master Architecture & Layer Specs (ARCHITECTURE)](#1-master-architecture--layer-specs)
2. [Executable Pipeline Runbook (PIPELINE)](#2-executable-pipeline-runbook)
3. [MSM Input Features Specification (FEATURES)](#3-msm-input-features-specification)
4. [Configuration Reference (CONFIGURATION)](#4-configuration-reference)
5. [Interfaces & Data Contracts (INTERFACES)](#5-interfaces--data-contracts)
6. [Validation & Acceptance Protocol (VALIDATION)](#6-validation--acceptance-protocol)
7. [Phased Implementation Roadmap (IMPLEMENTATION_PLAN)](#7-phased-implementation-roadmap)
8. [Architecture Decision Records (DECISIONS)](#8-architecture-decision-records)

---

# 1. Master Architecture & Layer Specs

## 1.1 The 15 Architectural Fixes

| # | Fix | Description & Rationale | Source Evidence |
|---|---|---|---|
| F1 | **Dual-Stage Spatial Selection** | (a) Keypoint ANMS (SSC variant) *before* matching for sparse matchers; (b) match-level grid/coverage selection *after* matching for all matchers. | Supplementary research §1; LoFTR-SPP |
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

## 1.2 Master Architecture Diagram

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
```

---

# 2. Executable Pipeline Runbook

## 2.1 Stage Table

| Stage | Script / Module | Input | Output | Gate |
|---|---|---|---|---|
| S0 Setup | Environment config | - | ASP env + ISISDATA | ASP active |
| S1 Ingest | `scripts/ingest.py` | Raw PDS4 zips | `.cub` + `products.jsonl` | `spiceinit` exit 0 |
| S2 Pairs | `scripts/build_pairs.py` | `products.jsonl` | `manifest.jsonl` + Crops | Overlap $\ge 0.5$ |
| S3 Preprocess | `scripts/preprocess.py` | Manifest record | `data/processed/<pair_id>/` | Mask 5–30% |
| S4.5 MSM | `src/selector/model.py` | PairRecord + `meta.json` | `selector.json` | Routing reason set |
| S4 Match | `src/matching/` | Patches + Routing | `matches_raw.json` | Candidates $\ge 150$ |
| S5 Select | `src/selection/` | `matches_raw.json` | `matches_selected.json` | Coverage $\ge 0.60$ |
| S6 Verify | `src/registration/` | Selected matches | `geometry.json` | Inlier ratio $\ge 0.05$ |
| S7 Refine | `src/refinement/` | Inliers + Patches | `matches_refined.json` | Refined $\ge 70\%$ |
| S8 Products | `scripts/register.py` | Refined matches | GeoTIFF / CSV / GCP | Warp valid $\ge 90\%$ |
| S9 Evaluate | `src/evaluation/` | Pipeline outputs | `leaderboard.csv` | Leakage audit passes |

---

# 3. MSM Input Features Specification

| Feature Name | Data Type | Range / Domain | Purpose & Extraction Logic |
|---|---|---|---|
| `sensor_pair_enc` | int | {0: OHRC-NAC, 1: TMC-WAC, 2: IIRS-WAC} | Encoded sensor modality pair |
| `gsd_ratio` | float | $(0, 1.0]$ | Ratio $\text{GSD}_{\text{source}} / \text{GSD}_{\text{reference}}$ |
| `latitude_abs` | float | $[0.0^\circ, 90.0^\circ]$ | Absolute latitude of bounding box centroid |
| `delta_solar_azimuth` | float | $[0.0^\circ, 180.0^\circ]$ | $|\text{Azimuth}_{\text{src}} - \text{Azimuth}_{\text{ref}}|$, clamped |
| `terrain_class_enc` | int | {0: highland, 1: maria, 2: polar, 3: mixed} | Encoded terrain type from manifest |
| `crater_density` | float | $[0, \infty)$ craters/Mpx | $\log(1 + \text{crater\_density})$ |
| `masked_fraction` | float | $[0.0, 1.0]$ | Fraction of image suppressed by validity mask |
| `overlap_fraction` | float | $(0.0, 1.0]$ | Spatial overlap fraction between image patches |
| `src_texture_contrast` | float | $[0, \infty)$ DN | Mean local standard deviation in $8\times 8$ windows (source) |
| `ref_texture_contrast` | float | $[0, \infty)$ DN | Mean local standard deviation in $8\times 8$ windows (reference) |
| `src_mean_gradient` | float | $[0, \infty)$ DN/px | Mean Sobel gradient magnitude (source) |
| `ref_mean_gradient` | float | $[0, \infty)$ DN/px | Mean Sobel gradient magnitude (reference) |
| `tile_count` | int | $[1, \sim 100]$ | Count of non-discarded tiles post-reconciliation |

---

# 4. Configuration Reference

```yaml
# configs/msm.yaml
msm:
  enabled: false                      # Toggle for production mode
  model_path: "models/msm_v1.pkl"
  model_stats_path: "models/msm_v1_stats.json"
  model_version: "msm_v1"

  tau_high: 0.65                      # Single matcher execution threshold
  tau_low: 0.40                       # Fallback matcher threshold

  hard_rules:
    crater_density_gate:
      enabled: true
      tau_c: 5.0                      # Craters / Mpx threshold
    gpu_gate:
      enabled: true
      check_at_startup: true
    iirs_track_gate:
      enabled: true

  fallback:
    on_model_load_error: "benchmark_mode"
    on_feature_extraction_error: "benchmark_mode"
    on_s4_gate_failure: "sift"
    log_all_fallback_events: true
```

---

# 5. Interfaces & Data Contracts

```python
@dataclass
class SelectorResult:
    pair_id:              str
    selected_matcher:     str         # "sift" | "rift2" | "lightglue" | "crater"
    confidence:           float       # max probability
    fallback_matcher:     str         # second choice matcher
    all_probs:            dict        # probability distribution over matchers
    routing_reason:       str         # e.g., "high_confidence", "low_confidence_fallback"
    matchers_to_run:      list[str]   # list passed to L2 engine
    hard_rules_applied:   list[str]
    selector_version:     str
    feature_vector_hash:  str
```

---

# 6. Validation & Acceptance Protocol

The selector must pass **8 Acceptance Criteria (AC1–AC8)** on held-out test data prior to enabling `msm.enabled: true`:

1. **AC1 Selector Accuracy**: $\ge 70.0\%$ (Matches oracle best matcher).
2. **AC2 Top-2 Accuracy**: $\ge 85.0\%$ (Oracle best is in top 2 choices).
3. **AC3 Mean RMSE Degradation**: $\le +0.10\text{ px}$ vs oracle best.
4. **AC4 Max Single-Pair RMSE Degradation**: $\le +0.50\text{ px}$.
5. **AC5 Runtime Reduction**: $\ge 50.0\%$ execution time savings.
6. **AC6 Fallback Rate**: $\le 20.0\%$ full-mode fallback rate.
7. **AC7 Feature Importance**: Non-trivial (top 5 features have $>0$ gain).
8. **AC8 Leakage Audit**: Exit Code 0 (`leakage_audit --check-msm`).

---

# 7. Phased Implementation Roadmap

- **Phase P5.5.0**: Prerequisite checks ($\ge 50$ benchmarked pairs).
- **Phase P5.5.1**: Data layer & `meta.json` feature computation.
- **Phase P5.5.2**: `FeatureVector` extraction module (`src/selector/features.py`).
- **Phase P5.5.3**: Configuration & schema validation (`configs/msm.yaml`).
- **Phase P5.5.4**: `MatcherSelector` engine & routing logic (`src/selector/model.py`).
- **Phase P5.5.5**: Integration into `scripts/benchmark.py`.
- **Phase P5.5.6**: Training script implementation (`scripts/train_msm.py`).
- **Phase P5.5.7**: MSM Evaluation module (`src/evaluation/msm_eval.py`).
- **Phase P5.5.8**: Validation verification & production activation.

---

# 8. Architecture Decision Records

- **DEC-001**: Pluggable Matcher Engine Architecture.
- **DEC-002**: Two-Stage Spatial Selection (ANMS + Coverage Grid).
- **DEC-003**: Geodetic Control Network & Padded Reference Patching.
- **DEC-004**: Matcher Selection Model (MSM) Position in Pipeline (L1.5).
- **DEC-005**: LightGBM as the MSM Classifier Framework.
- **DEC-006**: Dual Confidence Threshold Routing & Escalation Fallback Policy.
- **DEC-007**: Strict Geo-Cell Disjointness for MSM Training (F15 Leakage Rule).
- **DEC-008**: Operational Gate for MSM Production Activation.
