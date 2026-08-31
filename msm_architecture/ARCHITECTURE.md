# SIH26166 — ARCHITECTURE (Master System Architecture, v2.0)

**Multi-modal, Sun-angle & scale-invariant image correspondence using Chandrayaan-2 optical images
(OHRC / TMC-2 / IIRS) vs LRO (NAC / WAC) reference**

**v2.0 change vs v1.0:** A lightweight **Matcher Selection Model (MSM)** has been inserted as layer
**L1.5** between Preprocessing (L1) and the Correspondence Engine (L2). The MSM uses cheap,
early-available pair features to predict which single matcher is most likely to perform best,
eliminating the need to run all four matchers on every pair in production. All other layers
(L0 through L7), the Matcher interface, the benchmark mode, and the existing fallback mechanism
are unchanged. See also: PIPELINE.md (execution), FEATURES.md (MSM inputs), CONFIGURATION.md
(config keys), INTERFACES.md (contracts), VALIDATION.md (acceptance criteria), DECISIONS.md
(rationale).

Basis: full analysis of all 15 research files in SIH_dump plus verified live data-source checks
(PRADAN/CHMAP, ASP docs, Lunar ODE, LROC archive).

Core principle: **benchmark-first, model-second.** The L7 harness decides the winning matcher per
stratum; the MSM is trained on that leaderboard output. Every matcher remains pluggable behind one
interface.

---

## 0. The 14 fixes applied to the draft architecture (unchanged from v1.0)

| # | Fix | Why (evidence) |
|---|-----|----------------|
| F1 | **Dual-stage spatial selection**: (a) keypoint-level ANMS (SSC variant) *before* matching for sparse matchers; (b) match-level grid/coverage-aware selection *after* matching for all matchers. | Supplementary research §1; LoFTR-SPP. |
| F2 | **Mandatory geometric sanity check on learned matchers**: in-image-domain bounds + one-to-one constraint. | HybridPhaseCorrelation: LoFTR produces out-of-image-domain matches. |
| F3 | **Hierarchical geometric model ladder**: similarity → affine → homography → tile-wise local models. | SIFT-IIRS-WAC, DESCA, HybridPhaseCorrelation. |
| F4 | **Do not overspend on preprocessing for classical matchers.** | Traditional-vs-DL: classical methods failed polar despite full preprocessing. |
| F5 | **Deep-shadow validity masking before matching.** | KAZE (PC limitation in textureless/deep shadow); CNSFM. |
| F6 | **If DESCA adopted: two-tier match set** (strict NNDR≈0.7 + loose NNDR≈1.0). | DESCA takeaway #5. |
| F7 | **Phase-correlation specifics**: Gaussian/Tukey apodization, Gaussian-pyramid, 2D paraboloid. | HybridPhaseCorrelation takeaway #7. |
| F8 | **Match count is never a quality proxy.** Every leaderboard row reports RMSE + inlier ratio + spatial coverage together. | Traditional-vs-DL takeaway #6. |
| F9 | **Leakage rule**: benchmark splits by disjoint geographic cells (10°×10°); split id in manifest. | Standard ML hygiene. |
| F10 | **Provenance rules**: never rename ISRO .img/.xml products; ASP ≥3.7.0 conda env; per-orbit-date CK kernels only. | ASP §8.15 (verified live). |
| F11 | **Matcher arbitration, not monoculture**: confidence-based fallback. | CNSFM+SuperGlue hybrid hypothesis. |
| F12 | **IIRS is a separate track.** | Supplementary research §7. |
| F13 | **Spatial Coverage is a first-class metric** (grid density std-dev / percent cells populated). | HybridPhaseCorrelation; LoFTR greedy point-merging. |
| F14 | **Crater/geometric branch is gated**: crater-density check first. | CNSFM: 72.3% SR at south pole vs 31.2% for best baseline — explicit failure in crater-sparse units. |
| **F15** | **[NEW v2.0] MSM training uses geo-cell-disjoint splits (same F9 rule) and is only activated after the full benchmark validates it.** | Prevents selector leakage from contaminating the benchmark that produced its labels. |

---

## 1. Master architecture diagram (v2.0)

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
 ║ L1.5  MATCHER SELECTION MODEL  [NEW — v2.0]                   ║
 ║  FeatureVector ← {sensor_pair, gsd_ratio, latitude_abs,       ║
 ║    delta_solar_azimuth, terrain_class, crater_density,        ║
 ║    masked_fraction, overlap_fraction, texture_contrast×2,     ║
 ║    mean_gradient×2, tile_count}  (13 features, see FEATURES)  ║
 ║  Model: LightGBM multi-class classifier (4 classes = matchers)║
 ║  → SelectorResult {selected_matcher, confidence, fallback,    ║
 ║                    all_probs, routing_reason}                  ║
 ║  Routing:                                                      ║
 ║    confidence ≥ τ_high (0.65) → run selected_matcher only     ║
 ║    confidence ∈ [τ_low, τ_high) → run fallback_matcher        ║
 ║    confidence < τ_low (0.40)  → run all matchers (safe mode)  ║
 ║  Hard rules: M3 blocked if crater_density < τ_c;              ║
 ║              M2 blocked if no GPU                              ║
 ║  Benchmark mode: selector bypassed; all matchers run (L7 data ║
 ║    collection and MSM training)                                ║
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
 └───────────────────────────────────────────────────────────────┘

 Side track (parallel, own frontend — F12):
   IIRS QUB → photometric correction → SIFT-class match vs WAC
   → sub-80 m RMSE (sub-pixel at 80 m GSD)
```

---

## 2. Layer specifications

### L0 — Data & Geometry Layer
*(unchanged from v1.0)*
- **Inputs**: PRADAN/CHMAP zips (.img + .xml, PDS4; IIRS = QUB). Original filenames preserved (F10).
- **Processing**:
  1. Unzip → data/raw/ (keep ISRO naming).
  2. isisimport → .cub; spiceinit/CSM via ASP-bundled ISIS + ALE + USGSCSM. CK kernels fetched per orbit date only.
  3. Label/XML parser → per-product metadata: corner lat/lon footprint, solar incidence & azimuth, UTC, GSD, product_id.
  4. Padded bbox = footprint expanded 2–5× pointing uncertainty.
  5. Reference acquisition: Lunar ODE bbox query for NAC strips; GDAL crop of local WAC 643 nm mosaic.
- **Output (data contract)**: PairRecord (see §4) appended to data/pairs/manifest.jsonl.

### L1 — Preprocessing
*(unchanged from v1.0; now additionally writes L1 meta.json fields consumed by MSM)*
- **Shadow/validity mask (F5)**: dark + flat + cast pixel rules; per-pair mask.
- **Radiometric normalization**: percentile clip (2/98) → min-max → mean/std transfer.
- **Sensor branches**: OHRC→NAC = CLAHE+PCA; TMC-2→WAC = histogram-match+CLAHE. Learned matchers skip (F4).
- **Tiling**: overlapping tiles; per-tile processing.
- **GSD reconciliation**: pyramid resampling.
- **[NEW v2.0] L1 meta.json extended fields** (computed cheaply during preprocessing, consumed by MSM):
  - `masked_fraction` — fraction of pixels flagged by validity mask
  - `src_texture_contrast` — mean local contrast (std in 8×8 windows) of source patch
  - `ref_texture_contrast` — same for reference patch
  - `src_mean_gradient` — mean Sobel gradient magnitude of source patch (after normalization)
  - `ref_mean_gradient` — same for reference patch
  - `tile_count` — number of valid (non-discarded) tiles

---

### L1.5 — Matcher Selection Model (NEW — v2.0)

This layer is the only new architectural component. It sits between L1 and L2 and operates in two modes:

**Mode A — Benchmark mode** (used during P0–P5, MSM training, and any time `msm.enabled: false` in config):
- Layer is a transparent pass-through.
- All matchers run as before, gated only by the existing hard rules (M3 crater-density gate, M2 GPU gate).
- L7 leaderboard data is collected for MSM training.

**Mode B — Production / MSM mode** (used after MSM passes VALIDATION.md acceptance criteria):
- Extracts FeatureVector from PairRecord + L1 meta.json (zero additional I/O).
- Runs LightGBM inference (< 1 ms on CPU).
- Emits SelectorResult (see §4 and INTERFACES.md).
- Routes to a single matcher (or triggers safe fallback — see below).

#### Routing policy (confidence thresholds)

```
τ_high = 0.65   # (configurable; see CONFIGURATION.md)
τ_low  = 0.40   # (configurable)

confidence = SelectorResult.confidence   # = max(all_probs)

if confidence >= τ_high:
    run selected_matcher only
    routing_reason = "high_confidence"

elif confidence >= τ_low:
    run fallback_matcher  (= argmax2(all_probs), after hard-rule masking)
    routing_reason = "low_confidence_fallback"
    # M0 still runs in parallel if it is not already selected/fallback
    # (M0 always runs per the existing design; this is not a new cost)

else:  # confidence < τ_low
    run all matchers  (revert to benchmark mode for this pair)
    routing_reason = "very_low_confidence_full_mode"
    # pair is logged in results/msm_fallback.jsonl
```

#### Hard-rule overrides (applied before confidence check)

These rules are identical to the existing arbitration policy (§5) and are applied by zeroing the
relevant probability and renormalizing:
- M3 (Crater): masked to 0 if `crater_density < τ_c` (default 5 craters/Mpx) in either image.
- M2 (LightGlue): masked to 0 if `gpu_available = false`.

After masking, the softmax probabilities are renormalized. Only then is `confidence` computed.

#### Training the MSM

| Aspect | Specification |
|---|---|
| **Label** | Winning matcher per pair = argmax of composite score: `0.5×(1/RMSE_norm) + 0.25×inlier_ratio + 0.25×spatial_coverage`, normalized per pair across all matchers that succeeded. |
| **Training pairs** | Only pairs where all applicable matchers produced a valid RegistrationResult. Minimum 50 labeled pairs before training is attempted. |
| **Feature extraction** | From PairRecord + L1 meta.json (already written; no re-processing). |
| **Model** | LightGBM multi-class classifier (`objective: multiclass`, `num_class: 4`). |
| **Split** | Same geo-cell split as the benchmark (F9 / F15). MSM train = benchmark train cells only. |
| **Hyperparameters** | Cross-validated on train split; final model retrained on all train pairs; defaults in `configs/msm.yaml`. |
| **Labeling script** | `scripts/train_msm.py --leaderboard results/leaderboard.csv --manifest data/pairs/manifest.jsonl --split train --out models/msm_v1.pkl` |

#### MSM benchmark (measured against all-matchers baseline)

See VALIDATION.md §2 for the full acceptance protocol. The MSM is only activated in production
after passing all acceptance criteria on the test split.

---

### L2 — Correspondence Engine (pluggable)
*(unchanged from v1.0, except it now receives a routing signal from L1.5)*

All matchers implement one interface (§3). In benchmark mode all run; in MSM production mode only
the routed matcher(s) run. M0 always acts as the implicit fallback:
- **M0 SIFT**: tiled SIFT → Lowe ratio 0.75 → RANSAC homography (baseline, always runs in benchmark).
- **M1 RIFT/RIFT2**: phase-congruency + MIM descriptor; multi-octave log-Gabor scale-space extension.
- **M2 SuperPoint + LightGlue**: pretrained, confidence scores feed L3/L4; F2 checks mandatory.
- **M3 Crater-geometry (CNSFM-style)**: gated by crater density (F14).

### L3 — Uniform Correspondence Optimization *(unchanged)*
1. Keypoint ANMS (SSC) pre-match for sparse matchers M0/M1 (F1a).
2. Confidence filter → grid partition → per-cell cap → coverage-aware greedy selection (F1b).
3. Output: filtered match set + coverage statistic.

### L4 — Geometric Verification & Model Estimation *(unchanged)*
- DEGENSAC/MAGSAC++; model ladder (F3); F2 checks; optional DESCA refinement (F6); GCP declustering.

### L5 — Sub-pixel Refinement *(unchanged)*
- Local NCC/phase correlation; Gaussian/Tukey window; 2D paraboloid peak fit (F7).

### L6 — Product Generation *(unchanged)*
- Registered GeoTIFF; match-points CSV/GCP; QC images.

### L7 — Evaluation Harness *(extended in v2.0)*
*(unchanged except for MSM-related outputs)*
- Leaderboard per (matcher × sensor-pair × stratum): RMSE, pct<1px, pct<0.5px, MedAE, inlier
  count/ratio, spatial coverage, runtime.
- Ground truth: manual tie-points + cross-method consistency + LOLA anchor.
- Leakage policy (F9): splits by disjoint 10°×10° cells.
- **[NEW v2.0] MSM benchmark outputs** (only when `--mode msm_eval` is passed):
  - `results/msm_benchmark.csv` — selector accuracy, top-2 accuracy, RMSE degradation,
    runtime reduction, fallback rate per test pair.
  - Aggregate metrics appended to `results/leaderboard.csv` under a `msm_*` matcher column.

---

## 3. Matcher interface contract (pluggability — unchanged)

```python
class Matcher(ABC):
    name: str   # registry key: sift | rift2 | lightglue | crater

    @abstractmethod
    def detect_and_describe(self, img, valid_mask=None) -> Keypoints:
        # returns xy, scales, orientations, descriptors; respects valid_mask (F5)
        ...

    @abstractmethod
    def match(self, kp_src, kp_ref, cfg) -> MatchSet:
        # returns index pairs + confidence in [0,1] per match
        ...

    def supports_semidense(self) -> bool:
        return False  # LoFTR-style matchers override match() to skip detection
```

See INTERFACES.md for the new `MatcherSelector` interface and all v2.0 data contracts.

---

## 4. Data contracts

### PairRecord (data/pairs/manifest.jsonl)
*(unchanged fields; `msm_split` field added in v2.0)*

```json
{
  "pair_id":         "ohr_20200827T0030__nac_M123456789",
  "source":          {"sensor": "OHRC", "product_id": "ch2_ohr_nrp_...", "path": "data/raw/..."},
  "reference":       {"sensor": "NAC",  "product_id": "M123456789",       "path": "data/reference/..."},
  "bbox_pad_deg":    [lon_min, lat_min, lon_max, lat_max],
  "illum": {
    "src_incidence": 0.0, "src_azimuth": 0.0,
    "ref_incidence": 0.0, "ref_azimuth": 0.0,
    "delta_azimuth": 0.0
  },
  "terrain_class":   "highland|maria|polar|mixed",
  "crater_density":  0.0,
  "overlap_fraction": 1.0,
  "gsd_ratio":       0.56,
  "geo_cell":        "20E_15S",
  "split":           "train|test",
  "msm_split":       "train|test"
}
```

`msm_split` defaults to the same value as `split`. It can be overridden independently to reserve
additional pairs as an MSM-only hold-out without disturbing the benchmark split.

### RegistrationResult (per pair, per matcher — results/pair_results/)
*(unchanged)*

```json
{
  "pair_id": "...", "matcher": "lightglue",
  "n_candidates": 4210, "n_inliers": 383, "inlier_ratio": 0.091,
  "model": "homography|tilewise_affine", "rmse_px": 0.38,
  "pct_lt_1px": 97.2, "pct_lt_0p5px": 84.1, "medae_px": 0.22,
  "spatial_coverage": 0.964, "grid_density_std": 0.41,
  "runtime_s": 1.9, "refined": true
}
```

### SelectorResult (NEW — results/pair_results/<pair_id>/selector.json)

```json
{
  "pair_id":          "ohr_20200827T0030__nac_M123456789",
  "selected_matcher": "lightglue",
  "confidence":       0.81,
  "fallback_matcher": "rift2",
  "all_probs":        {"sift": 0.05, "rift2": 0.14, "lightglue": 0.81, "crater": 0.00},
  "routing_reason":   "high_confidence|low_confidence_fallback|very_low_confidence_full_mode",
  "hard_rules_applied": ["crater_density_gate"],
  "selector_version": "msm_v1",
  "feature_vector_hash": "sha256:..."
}
```

### FeatureVector (in-memory; logged to results/pair_results/<pair_id>/msm_features.json)

See FEATURES.md for the complete feature specification, extraction method, and normalization.

---

## 5. Matcher arbitration policy (F11 — benchmark / no-MSM mode)
*(unchanged; used when MSM is disabled or confidence < τ_low)*

```
if crater_density >= threshold and terrain_class in {highland, polar}:
    primary = M3 (crater-geometry)
elif learned_confidence_ok and gpu_available:
    primary = M2 (LightGlue)
else:
    primary = M1 (RIFT)   # no-GPU, no-training illumination-robust default
always: M0 runs in parallel as baseline + fallback
if primary inlier_ratio < floor:
    fall back to M0 result for that pair (recorded in arbitration log)
```

---

## 6. Repository layout (v2.0 additions marked [NEW])

```
SIH/
├── architecture/
│   ├── ARCHITECTURE.md          [this file — v2.0]
│   ├── PIPELINE.md              [updated v2.0]
│   ├── FEATURES.md              [NEW]
│   ├── CONFIGURATION.md         [NEW]
│   ├── INTERFACES.md            [NEW]
│   ├── VALIDATION.md            [NEW]
│   ├── IMPLEMENTATION_PLAN.md   [NEW]
│   └── DECISIONS.md             [NEW]
├── configs/
│   ├── ohrc_nac.yaml
│   ├── tmc_wac.yaml
│   ├── iirs_wac.yaml
│   ├── matchers.yaml
│   └── msm.yaml                 [NEW]
├── data/
│   ├── raw/            # ISRO zips, original filenames (F10)
│   ├── calibrated/     # ISIS .cub after isisimport/spiceinit
│   ├── reference/      # NAC strips, WAC mosaic + crops
│   ├── pairs/          # manifest.jsonl
│   └── metadata/       # parsed labels, SPICE/kernel logs, GT tie-points
├── models/
│   └── msm_v1.pkl               [NEW — trained LightGBM selector]
├── src/
│   ├── ingest/
│   ├── preprocessing/
│   ├── geometry/
│   ├── matching/
│   │   ├── base.py
│   │   ├── sift.py
│   │   ├── rift.py
│   │   ├── lightglue.py
│   │   └── crater.py
│   ├── selection/      # anms.py, spatial.py (unchanged)
│   ├── selector/                [NEW]
│   │   ├── __init__.py
│   │   ├── features.py         # FeatureVector extraction
│   │   ├── model.py            # MatcherSelector: load/infer/route
│   │   └── train.py            # label generation + LightGBM training
│   ├── registration/
│   ├── refinement/
│   └── evaluation/
│       ├── metrics.py
│       ├── leaderboard.py
│       └── msm_eval.py         [NEW — MSM benchmark metrics]
├── scripts/
│   ├── build_pairs.py
│   ├── preprocess.py
│   ├── benchmark.py            [extended: --mode msm_eval]
│   ├── register.py
│   └── train_msm.py            [NEW]
├── notebooks/
├── results/
│   ├── leaderboard.csv
│   ├── pair_results/           # includes selector.json, msm_features.json [NEW]
│   ├── msm_benchmark.csv       [NEW]
│   ├── msm_fallback.jsonl      [NEW]
│   └── figures/
└── app/
```

---

## 7. Implementation phases (v2.0)

*(Phases P0–P5 unchanged; P5.5 is new)*

1. **P0 — Setup**: repo scaffold, config schema, ASP 3.7.0 conda env.
2. **P1 — Dataset**: download pilot set, label parser, build_pairs.py, geo-cell splits.
3. **P2 — Baseline**: M0 SIFT end-to-end; manual GT on 5 pairs; leaderboard v0.
4. **P3 — Scale**: expand to 50 pairs; GT on 15–20 pairs.
5. **P4 — Matchers**: M1, M2, M3 behind the interface; L3 selection; L5 refinement; leaderboard v1.
6. **P5 — Arbitration + products**: full arbitration policy; GeoTIFF deliverables; final leaderboard.
7. **P5.5 — MSM (NEW)**: extend L1 meta.json with MSM features; run `train_msm.py`; benchmark selector; enable production mode only if VALIDATION.md acceptance criteria are met.
8. **P6 — IIRS track (F12)**: photometric correction → SIFT-class vs WAC.
9. **P7 — App/UI**: only after the pipeline is reliable.

---

## 8. Risk register (v2.0 additions)

| Risk | Likelihood | Mitigation |
|---|---|---|
| PRADAN download friction | Medium | Start with 3 verified products from ASP example |
| spiceinit/camera-model issues on Chandrayaan-2 | Medium | Use ASP ≥3.7.0 bundled stack |
| Learned matcher domain gap | Medium | F2 checks + M0 fallback per pair |
| RIFT runtime blow-up on OHRC strips | High | Tile-level matching + runtime as first-class metric |
| Crater-sparse terrain failure (M3) | High (in mare) | F14 gating by crater density |
| Ground-truth scarcity | Certain | 3-layer GT protocol |
| Polar / >±55° geometry | High | F3 tile-wise models; report per-stratum |
| GPU constraints | Medium | M1/M0 are CPU-only; M2 optional |
| **[NEW] MSM training set too small (<50 pairs)** | Medium | **Fall back to rule-based arbitration policy (§5) until 50 labeled pairs exist** |
| **[NEW] MSM overfits to train strata** | Low–Medium | **Geo-cell-disjoint split (F9/F15); feature importance audit; RMSE degradation threshold ≤ +0.10 px enforced** |
| **[NEW] MSM adds latency** | Very Low | **LightGBM inference < 1 ms CPU; feature extraction reuses already-computed L1 stats** |
| **[NEW] MSM mis-routes in low-confidence pairs** | Low | **τ_low fallback reverts to full multi-matcher mode; every routing decision logged in selector.json** |

---

## 9. Evidence map (decision → research file)

| Decision | Source file(s) |
|---|---|
| Benchmark-first, pluggable engine | Traditional_vs_DeepLearning_FeatureMatching.md |
| ANMS dual-stage, spatial coverage metric | SIH26166_supplementary_research(2).md §1; LoFTR_IMC21.md |
| DEGENSAC / geometric sanity | LoFTR_IMC21.md; HybridPhaseCorrelation(2026).md |
| Model ladder + tile-wise, latitude limits | SIFT-IIRS-WAC_extracted.md; DESCA(Dr.Sourabh).md; HybridPhaseCorrelation(2026).md |
| Preprocessing must not substitute for robust matching | Traditional_vs_DeepLearning_FeatureMatching.md |
| Shadow masking, PC limits | KAZE(2026).md; CNSFM_Crater_Neighborhood_Matching.md |
| Phase-correlation refinement recipe | HybridPhaseCorrelation(2026).md |
| DESCA two-tier initialization | DESCA(Dr.Sourabh).md |
| Crater branch + gating + pole numbers | CNSFM_Crater_Neighborhood_Matching.md |
| Learned matcher primary, day-night evidence | SuperGlue.md; Traditional_vs_DeepLearning_FeatureMatching.md |
| IIRS separate track | SIH26166_supplementary_research(2).md §7 |
| Data layer, provenance, ASP/ISIS | ASP §8.15 (verified live); MoonMetaSync_extracted.md |
| Radiometric normalization as helper only | Radiometric_Normalization_Analysis.md |
| SIFT baseline architecture | SIFT-IIRS-WAC_extracted.md |
| RIFT scale-space extension | RIFT_extracted.md |
| Metric suite | HybridPhaseCorrelation(2026).md; RIFT_extracted.md |
| **[NEW] MSM: LightGBM choice, feature set, τ thresholds** | **See DECISIONS.md** |
| **[NEW] MSM: geo-cell leakage prevention (F15)** | **Standard ML hygiene; F9 encoded in data contract** |

---

## 10. Why this architecture beats the alternatives (unchanged from v1.0)

*(See §10 of the original BEST_ARCHITECTURE_KNOWN (1).md — the comparison table is not repeated
here because the MSM does not change which matchers exist or how they perform; it only changes
which one runs per pair in production.)*

The MSM is an **operational efficiency layer**, not an accuracy layer. Its correctness is
validated empirically via the benchmark in VALIDATION.md §2 before it is activated.

---

## 11–18. Stage algorithms, matcher spec cards, dataset engineering, worked examples, config examples, testing, compute budget, deliverables mapping

*(All sections §11–§18 from BEST_ARCHITECTURE_KNOWN (1).md v1.0 are unchanged and remain
authoritative. The MSM adds the following to two sections:)*

### Addition to §11: Stage algorithms — L1.5 MSM routing (pseudocode)

```python
# src/selector/model.py
def route(pair_record: PairRecord, l1_meta: L1Meta, cfg: MSMConfig) -> SelectorResult:
    fv = extract_features(pair_record, l1_meta)          # FEATURES.md §2
    probs = model.predict_proba([fv.to_array()])[0]      # LightGBM, shape (4,)
    probs = apply_hard_rules(probs, pair_record, cfg)    # zero + renormalize
    selected_idx = np.argmax(probs)
    confidence   = probs[selected_idx]
    fallback_idx = np.argsort(probs)[-2]                 # second-highest after masking

    if confidence >= cfg.tau_high:
        routing = "high_confidence"
        matchers_to_run = [MATCHER_NAMES[selected_idx]]
    elif confidence >= cfg.tau_low:
        routing = "low_confidence_fallback"
        matchers_to_run = [MATCHER_NAMES[fallback_idx]]
    else:
        routing = "very_low_confidence_full_mode"
        matchers_to_run = ["sift", "rift2", "lightglue", "crater"]  # all (with hard gates)

    return SelectorResult(
        selected_matcher=MATCHER_NAMES[selected_idx],
        confidence=float(confidence),
        fallback_matcher=MATCHER_NAMES[fallback_idx],
        all_probs=dict(zip(MATCHER_NAMES, probs.tolist())),
        routing_reason=routing,
        matchers_to_run=matchers_to_run,
    )
```

### Addition to §16: Testing & acceptance — MSM layer

| Test | Acceptance |
|---|---|
| Unit: `route()` on synthetic FeatureVector | Returns valid SelectorResult; all_probs sum to 1.0 ± 1e-6; hard rules zeroed correctly |
| Unit: hard-rule masking | M3 prob = 0 when crater_density < τ_c; M2 prob = 0 when gpu=False; remaining probs renormalize |
| Integration: `train_msm.py` on pilot data | Model file written; no exception; feature importance non-trivial (all top-5 features have >0 importance) |
| MSM leakage audit | `leakage_audit --check-msm` exits 0: zero shared geo-cells between MSM train/test |
| MSM benchmark | Selector accuracy ≥ 70% AND RMSE degradation ≤ +0.10 px on test split |
| MSM fallback | When confidence < τ_low: all applicable matchers run; selector.json written with `routing_reason: very_low_confidence_full_mode` |

---

**Bottom line (v2.0):** The MSM layer is a production efficiency addition that leverages the
leaderboard already produced by P5 to reduce per-pair runtime to ~1/4 in the common case, while
retaining a safe fallback for uncertain or novel pair configurations. It does not change the
matchers, the evaluation methodology, or the accuracy ceiling of the pipeline — it only determines
which matcher runs first (and often only). Its own accuracy is held to the same empirical,
leakage-free standard as everything else in this architecture.

*Companion docs: PIPELINE.md (execution runbook), FEATURES.md, CONFIGURATION.md, INTERFACES.md,
VALIDATION.md, IMPLEMENTATION_PLAN.md, DECISIONS.md.*
