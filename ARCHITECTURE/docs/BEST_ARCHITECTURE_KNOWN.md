# SIH26166 — BEST ARCHITECTURE KNOWN (Master System Architecture, FINAL v1.0)

**Multi-modal, Sun-angle & scale-invariant image correspondence using Chandrayaan-2 optical images (OHRC / TMC-2 / IIRS) vs LRO (NAC / WAC) reference**

Basis: full analysis of all 15 research files in SIH_dump (CNSFM, DESCA, HybridPhaseCorrelation, KAZE/I-KAZE, LoFTR-IMC21, MoonMetaSync, NASA sub-pixel co-registration, RIFT, Radiometric Normalization, SIFT-IIRS-WAC, SuperGlue, Traditional-vs-DL benchmark + 3 ai_suggestions docs) and verified live data-source checks (PRADAN/CHMAP, ASP docs, Lunar ODE, LROC archive).

Core principle: **benchmark-first, model-second.** The harness decides the winning matcher; the architecture makes every matcher pluggable behind one interface.

---

## 0. The 14 fixes applied to the draft architecture (each traces to a research file)

| # | Fix | Why (evidence) |
|---|-----|----------------|
| F1 | **Dual-stage spatial selection**: (a) keypoint-level ANMS (SSC variant) *before* matching for sparse matchers (detect → ANMS → describe → match); (b) match-level grid/coverage-aware selection *after* matching for all matchers. | Supplementary research §1 (ANMS belongs pre-match); LoFTR-SPP merges on matches post-hoc. Draft placed ANMS only after matching. |
| F2 | **Mandatory geometric sanity check on learned matchers**: in-image-domain bounds + one-to-one constraint on every learned match set. | HybridPhaseCorrelation: LoFTR produces out-of-image-domain, unstable extrapolated matches — the check layer is mandatory, not optional. |
| F3 | **Hierarchical geometric model ladder**: similarity → affine → homography → tile-wise local models. A single global homography/polynomial breaks beyond ±55° latitude and on high-relief terrain. | SIFT-IIRS-WAC (exponential error growth beyond ±55°), DESCA (affine fails on mountains), HybridPhaseCorrelation (terrain relief governs the accuracy ceiling). |
| F4 | **Do not overspend on preprocessing for classical matchers.** Heavy CLAHE/shadow-normalization branches demonstrably fail to rescue SIFT/AKAZE/ASIFT at the poles. Preprocessing helps; the matcher must carry illumination robustness. | Traditional-vs-DL: classical methods failed polar + SAR-SELENE despite the full preprocessing pipeline. |
| F5 | **Deep-shadow validity masking before matching.** Compute a shadow/low-information mask from solar geometry + local variance; exclude masked pixels rather than forcing matches. Phase congruency cannot invent structure in zero-information regions. | KAZE paper (PC limitation in textureless/deep shadow); CNSFM (failures = loss of shared visible terrain under near-180° azimuth change). |
| F6 | **If DESCA is adopted for refinement, replicate its two-tier match set** (strict NNDR≈0.7 for initialization + loose NNDR≈1.0 as evaluation pool). Random initialization demonstrably diverges to zero correct matches. | DESCA paper, takeaway #5. |
| F7 | **Phase-correlation specifics**: Gaussian/Tukey apodization windows (never Blackman), Gaussian-pyramid coarse-to-fine, 2D paraboloid peak fit for sub-pixel. | HybridPhaseCorrelation takeaway #7. |
| F8 | **Match count is never a quality proxy** (AKAZE counter-example). Every leaderboard row reports RMSE + inlier ratio + spatial coverage together, on held-out points. | Traditional-vs-DL takeaway #6. |
| F9 | **Leakage rule formalized**: benchmark splits by disjoint geographic cells (10°×10° lon/lat), never random pair splits. Split assignment stored in the pair manifest. | Standard ML hygiene; encoded in the data contract. |
| F10 | **Provenance rules**: never rename ISRO .img/.xml products (breaks isisimport); use ASP ≥3.7.0 conda env (bundled ISIS 10.0.0 + ALE + USGSCSM with Chandrayaan-2 camera fixes incl. TMC-2 fore/aft); fetch only per-orbit-date CK kernels (e.g. ch2_att_27Jul2020_04Sep2020_v1.bc), not the full 200 GB set. | ASP §8.15 Chandrayaan-2 example (verified live). |
| F11 | **Matcher arbitration, not monoculture**: terrain-adaptive + confidence-based fallback (learned → classical when learned confidence is low; crater branch gated by crater density). | CNSFM (fails on crater-sparse mare/melt ponds); CNSFM+SuperGlue hybrid hypothesis; user fallback strategy. |
| F12 | **IIRS is a separate track**: photometric correction (incidence/emission/phase) → SIFT-class registration vs WAC; accuracy target = sub-pixel relative to 80 m GSD (sub-80 m RMSE). Never force hyperspectral through the panchromatic frontend. | Supplementary research §7; KAZE paper (large modality gap = optical-vs-SAR-like failure mode). |
| F13 | **Spatial Coverage is a first-class metric** (grid density std-dev / percent cells populated) alongside RMSE, percent<1px, percent<0.5px, MedAE, inlier count/ratio, runtime. | Problem statement requires uniform distribution; HybridPhaseCorrelation metric suite; LoFTR greedy point-merging. |
| F14 | **Crater/geometric branch is gated**: crater-density check first; branch activates only in crater-rich terrain; detector (YOLOv9 transfer / DeepMoon / TMC-2 U-Net) fine-tuned per sensor. | CNSFM: 72.3 percent SR at the lunar south pole vs 31.2 percent for the best baseline — but explicit failure in crater-sparse units. |

---

## 1. Corrected master architecture (diagram)

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
 │  GDAL crop of local WAC mosaic                                │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼  Pair = source patch + reference patch + meta
 ┌───────────────────────────────────────────────────────────────┐
 │ L1  PREPROCESSING                                             │
 │  shadow/validity mask (F5) → radiometric normalization        │
 │  (percentile clip + stat transfer) → sensor branch:           │
 │    OHRC→NAC: CLAHE        TMC-2→WAC: histogram-match + CLAHE  │
 │    (learned matchers skip heavy branches — F4)                │
 │  tiling for scene heterogeneity; GSD reconciliation pyramid   │
 └──────────────────────────┬────────────────────────────────────┘
                            ▼
 ┌───────────────────────────────────────────────────────────────┐
 │ L2  CORRESPONDENCE ENGINE  (pluggable, benchmarked)           │
 │  M0 SIFT + ratio test          (baseline, always runs)        │
 │  M1 RIFT/RIFT2 (PC + MIM)      (classical illumination-robust)│
 │  M2 SuperPoint + LightGlue     (learned, GPU)                 │
 │  M3 Crater-geometry branch     (gated by crater density, F14) │
 │  Interface: detect/describe/match → kpts, matches, scores     │
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
 │  RMSE (held-out), percent<1px, percent<0.5px, MedAE, inlier   │
 │  count/ratio, spatial coverage, runtime; precision/recall     │
 │  where GT exists → leaderboard → arbitration policy  (F11)    │
 └───────────────────────────────────────────────────────────────┘

 Side track (parallel, own frontend — F12):
   IIRS QUB → photometric correction → SIFT-class match vs WAC
   → sub-80 m RMSE (sub-pixel at 80 m GSD)
```

---

## 2. Layer specifications

### L0 — Data & Geometry Layer
- **Inputs**: PRADAN/CHMAP zips (.img + .xml, PDS4; IIRS = QUB). Original filenames preserved (F10).
- **Processing**:
  1. Unzip → data/raw/ (keep ISRO naming, e.g. ch2_ohr_nrp_20200827T0030107497_d_img_d18).
  2. isisimport → .cub; spiceinit / CSM via ASP-bundled ISIS + ALE + USGSCSM. CK kernels fetched per orbit date only.
  3. Label/XML parser → per-product metadata: corner lat/lon footprint, solar incidence & azimuth, UTC, GSD, product id.
  4. Padded bbox = footprint expanded 2–5× pointing uncertainty (metadata is a search window, never the final answer).
  5. Reference acquisition: Lunar ODE bbox query for NAC strips; GDAL crop of local WAC 643 nm mosaic (or Moon Trek WMTS) for WAC patches.
- **Output (data contract)**: PairRecord (see §4) appended to data/pairs/manifest.jsonl.
- **Failure modes**: stale/missing CK kernels (spiceinit fails) → kernel-date check; footprint metadata off by ~km → padding rule; NAC strip not covering the full OHRC footprint → allow partial overlap, record overlap fraction in manifest.

### L1 — Preprocessing
- **Shadow/validity mask (F5)**: from solar incidence/azimuth + local variance threshold; exported as per-pair mask; all downstream stages respect the mask.
- **Radiometric normalization**: percentile clip (2/98) → min-max → mean/std transfer toward reference statistics. Cheap, detector-agnostic.
- **Sensor branches** (validated in literature): OHRC→NAC = CLAHE → inversion → morphological dilation → PCA (Traditional-vs-DL). TMC-2→WAC = histogram match + CLAHE (our design — TMC-2 is the confirmed research gap; treat the branch as experimental and A/B it). Learned matchers M2/M3: minimal preprocessing only (F4).
- **Tiling**: overlapping tiles (tiles smaller than half the grid discarded); per-tile processing handles crater-vs-mare scene heterogeneity.
- **GSD reconciliation**: pyramid resampling of the higher-GSD image; keep a native-resolution path as a config variant (native-scale RMSE reported in reference pixels).

### L2 — Correspondence Engine (pluggable)
All matchers implement one interface (§3). The registry is config-driven; adding a matcher = new class + config entry. M0 always runs (fallback + baseline). Runtime recorded per pair.
- **M0 SIFT**: tiled SIFT → Lowe ratio 0.75 → RANSAC homography (validated architecture from SIFT-IIRS-WAC; ~73 m RMSE precedent on IIRS↔WAC).
- **M1 RIFT/RIFT2**: phase-congruency detection + MIM descriptor; add multi-octave log-Gabor scale-space (our contribution — RIFT lacks scale invariance); rotation via multi-MIM candidates. Known risks: 15–30× runtime, one documented total failure → the benchmark decides.
- **M2 SuperPoint + LightGlue**: pretrained, no lunar training data needed; confidence scores feed L3/L4. Mandatory F2 checks.
- **M3 Crater-geometry (CNSFM-style)**: detector (YOLOv9 transfer / DeepMoon / TMC-2 U-Net) → CNSF construction (crater + K-NN topology) → similarity-invariant geometric matching → MCR-style structural outlier removal. **Gated** by crater-density check (F14).

### L3 — Uniform Correspondence Optimization (the novelty component)
1. **Keypoint ANMS (F1a)** — SSC variant (BAILOOL/ANMS-Codes), applied pre-match for sparse matchers M0/M1 so detection itself is spatially uniform.
2. **Confidence filter** — drop matches below threshold τ (matcher-specific; M2 scores come from LightGlue).
3. **Grid partition** — N×N cells over the source image.
4. **Per-cell cap** — keep max-N highest-confidence matches per cell.
5. **Coverage-aware greedy selection** — target budget K: greedily admit matches maximizing cell coverage (bisection on threshold, LoFTR-SPP style, with one-to-one conflict resolution added).
- **Output**: filtered match set + coverage statistic; report before/after spatial distribution (F13).

### L4 — Geometric Verification & Model Estimation
- **DEGENSAC** (degeneracy-aware RANSAC; handles flat lunar terrain) or MAGSAC++. 0.5–3.0 px reprojection threshold tuned per pair GSD.
- **Model ladder (F3)**: fit similarity → affine → homography in order; accept the simplest model whose inlier RMSE is acceptable; else fall back to **tile-wise local models** (per-tile affine/homography, blended) for high-relief / beyond ±55° latitude scenes.
- **F2 checks**: all matches inside both image domains; one-to-one; residuals re-checked post-fit.
- **Optional DESCA refinement (F6)**: two-tier match sets (strict 0.7 / loose 1.0 NNDR), DE affine refinement — only where initial geometry is affine-like; skip on strong relief.
- **GCP declustering**: minimum spacing (15–20 px scaled by GSD), grid-cell keep-nearest-center; Z-score residual filter (needs >20 GCPs) — reused verbatim from SIFT-IIRS-WAC.

### L5 — Sub-pixel Refinement (F7)
- Per-match local patch: normalized cross-correlation or phase-only correlation in a search window around the coarse match.
- Gaussian/Tukey apodization (never Blackman); multi-scale pyramid; integer peak → 2D paraboloid fit → sub-pixel offset; reject refinements with low peak sharpness.
- Result: match list upgraded to sub-pixel coordinates; RMSE re-computed after refinement to quantify the gain (reported as its own metric).

### L6 — Product Generation
- Warp source with the fitted model(s) → **registered GeoTIFF** on the reference grid.
- **Match-points product**: CSV + GCP list (pixel coords in both images + lon/lat via reference georeferencing).
- QC artifacts: checkerboard overlay, side-by-side with match overlay, residual heat map.

### L7 — Evaluation Harness (the deliverable that picks the model)
- Leaderboard per (matcher × sensor-pair × stratum): RMSE on held-out manual checkpoints, percent<1px, percent<0.5px, MedAE, inlier count, inlier ratio, spatial coverage, runtime; precision/recall/matching-score where GT allows.
- Ground truth: (1) manual evenly-distributed tie points on 15–20 pairs (30–50 points each); (2) cross-method consistency adjudication; (3) independent geodetic anchor via ASP pc_align vs raw LOLA shots where available.
- **Leakage policy (F9)**: splits by disjoint 10°×10° cells; split id in the manifest; every report states its split.
- Output: results/leaderboard.csv + per-pair JSON → drives the default-matcher arbitration policy (F11).

---

## 3. Matcher interface contract (pluggability)

```python
class Matcher(ABC):
    name: str                      # registry key, e.g. sift, rift2, lightglue, crater

    @abstractmethod
    def detect_and_describe(self, img, valid_mask=None) -> Keypoints:
        # returns xy, scales, orientations, descriptors; respects valid_mask (F5)
        ...

    @abstractmethod
    def match(self, kp_src, kp_ref, cfg) -> MatchSet:
        # returns index pairs + confidence in [0,1] per match
        ...

    def supports_semidense(self) -> bool:
        return False               # LoFTR-style matchers override match() to skip detection
```

- Registry: configs/matchers.yaml maps name → class + default params.
- Adding a matcher = implement the interface + one config entry. No pipeline changes.
- Every matcher runs through the identical L3→L7 path; only L2 differs. This is what makes the benchmark fair.

## 4. Data contracts

### PairRecord (data/pairs/manifest.jsonl, one JSON object per line)

```json
{
  "pair_id": "ohr_20200827T0030__nac_M123456789",
  "source":  {"sensor": "OHRC", "product_id": "ch2_ohr_nrp_20200827T0030107497_d_img_d18", "path": "data/raw/..."},
  "reference": {"sensor": "NAC", "product_id": "M123456789", "path": "data/reference/..."},
  "bbox_pad_deg": [lon_min, lat_min, lon_max, lat_max],
  "illum": {"src_incidence": 0.0, "src_azimuth": 0.0, "ref_incidence": 0.0, "ref_azimuth": 0.0, "delta_azimuth": 0.0},
  "terrain_class": "highland|maria|polar|mixed",
  "crater_density": 0.0,
  "overlap_fraction": 1.0,
  "gsd_ratio": 0.56,
  "geo_cell": "20E_15S",
  "split": "train|test"
}
```

### RegistrationResult (per pair, per matcher — results/pair_results/)

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

## 5. Matcher arbitration policy (F11, decided by the leaderboard)

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

## 6. Repository layout

```
SIH/
├── BEST_ARCHITECTURE_KNOWN.md
├── configs/
│   ├── ohrc_nac.yaml
│   ├── tmc_wac.yaml
│   ├── iirs_wac.yaml
│   └── matchers.yaml
├── data/
│   ├── raw/            # ISRO zips, original filenames (F10)
│   ├── calibrated/     # ISIS .cub after isisimport/spiceinit
│   ├── reference/      # NAC strips, WAC mosaic + crops
│   ├── pairs/          # manifest.jsonl
│   └── metadata/       # parsed labels, SPICE/kernel logs
├── src/
│   ├── ingest/         # zip unpack, label/XML parser, footprint
│   ├── preprocessing/  # masks, normalization, sensor branches, tiling
│   ├── geometry/       # ASP/ISIS wrappers, bbox padding, ref query
│   ├── matching/
│   │   ├── base.py     # Matcher interface
│   │   ├── sift.py
│   │   ├── rift.py
│   │   ├── lightglue.py
│   │   └── crater.py
│   ├── selection/      # anms.py (SSC), spatial.py (grid+coverage)
│   ├── registration/   # DEGENSAC/MAGSAC, model ladder, declustering
│   ├── refinement/     # NCC/phase corr, paraboloid peak fit
│   └── evaluation/     # metrics, leaderboard, leakage audit
├── scripts/
│   ├── build_pairs.py
│   ├── preprocess.py
│   ├── benchmark.py
│   └── register.py
├── notebooks/
│   ├── 01_dataset.ipynb
│   ├── 02_baseline.ipynb
│   └── 03_analysis.ipynb
├── results/            # leaderboard.csv, pair_results/, figures/
└── app/                # LAST: UI only after pipeline is reliable
```

## 7. Implementation phases

1. **P0 — Setup**: repo scaffold, config schema, ASP 3.7.0 conda env (bundled ISIS/ALE/USGSCSM), per-date CK kernels.
2. **P1 — Dataset**: user registers on PRADAN/CHMAP; download pilot set (verified OHRC stereo pair + TMC-2 ortho/DEM from ASP example + stratified extras); label parser; build_pairs.py; first 10 pilot pairs verified visually in QuickMap; manifest + geo-cell splits.
3. **P2 — Baseline**: M0 SIFT end-to-end (L0→L7) on pilot pairs; manual tie-point GT on 5 pairs; leaderboard v0.
4. **P3 — Scale**: expand to 50 pairs across strata (latitude × Δ-azimuth × terrain); GT on 15–20 pairs.
5. **P4 — Matchers**: M1 RIFT (+scale-space) and M2 LightGlue behind the interface; L3 selection; L5 sub-pixel refinement; leaderboard comparison.
6. **P5 — Arbitration + products**: M3 gated crater branch; arbitration policy; registered GeoTIFF + match-points deliverables; final leaderboard = the answer to why-this-model.
7. **P6 — IIRS track (F12)**: photometric correction → SIFT-class vs WAC; sub-80 m RMSE target; separate module, own config.
8. **P7 — App/UI**: only after the pipeline is reliable.

## 8. Risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| PRADAN download friction / bandwidth | Medium | start with the 3 verified products from the ASP example; parallel Moon Trek WAC crops need no ISRO login |
| spiceinit/camera-model issues on Chandrayaan-2 | Medium | use ASP ≥3.7.0 bundled stack (fixes included); keep ISIS import logs for debugging |
| Learned matcher domain gap (lunar vs MegaDepth-style training) | Medium | F2 checks + M0 fallback per pair; fine-tune later only if benchmark shows the gap |
| RIFT runtime blow-up on OHRC strips | High | tile-level matching + GSD reconciliation first; benchmark runtime as first-class metric |
| Crater-sparse terrain failure (M3) | High (in mare) | F14 gating by crater density |
| Ground-truth scarcity | Certain | 3-layer GT protocol (manual tie points, cross-method consistency, LOLA anchor) |
| Polar / >±55° geometry | High | F3 tile-wise models; report per-stratum, never hide polar rows |
| GPU constraints during the hackathon | Medium | M1/M0 are CPU-only; M2 optional per-pair |

## 9. Evidence map (decision → research file)

| Decision in this architecture | Source file(s) |
|---|---|
| Benchmark-first, pluggable engine, common evaluation | Traditional_vs_DeepLearning_FeatureMatching.md |
| ANMS dual-stage selection, spatial coverage metric | SIH26166_supplementary_research(2).md §1; LoFTR_IMC21.md |
| DEGENSAC / geometric sanity layer | LoFTR_IMC21.md; HybridPhaseCorrelation(2026).md |
| Model ladder + tile-wise models, latitude limits | SIFT-IIRS-WAC_extracted.md; DESCA(Dr.Sourabh).md; HybridPhaseCorrelation(2026).md |
| Preprocessing must not substitute for robust matching | Traditional_vs_DeepLearning_FeatureMatching.md |
| Shadow masking, PC limits | KAZE(2026).md; CNSFM_Crater_Neighborhood_Matching.md |
| Phase-correlation refinement recipe | HybridPhaseCorrelation(2026).md |
| DESCA two-tier initialization | DESCA(Dr.Sourabh).md |
| Crater branch + gating + pole numbers | CNSFM_Crater_Neighborhood_Matching.md |
| Learned matcher primary candidate, day-night evidence | SuperGlue.md; Traditional_vs_DeepLearning_FeatureMatching.md |
| IIRS separate track, photometric correction, sub-80 m target | SIH26166_supplementary_research(2).md §7 |
| Data layer, provenance, ASP/ISIS specifics, product IDs | ASP §8.15 (verified live); MoonMetaSync_extracted.md; SIH26166_reference_mapping.md |
| Radiometric normalization as helper only | Radiometric_Normalization_Analysis.md |
| SIFT baseline architecture (tiled, ratio, RANSAC, declustering) | SIFT-IIRS-WAC_extracted.md |
| RIFT scale-space extension as novelty | RIFT_extracted.md |
| Metric suite (RMSE, percent<1px, MedAE, inlier ratio, coverage) | HybridPhaseCorrelation(2026).md; RIFT_extracted.md |

---


---

# DEEP-DIVE ADDENDUM (v1.1)

## 10. Why this architecture beats the alternatives (honest comparison)

Scored against the eight hard requirements of SIH26166, using the numbers from the research files. ✗ = fails, ~ = partial/unproven, ✓ = works.

| Requirement | A: SIFT+RANSAC only | B: LoFTR trained from scratch | C: ASP-only (stereo/DEM route) | D: CNSFM crater only | E: RIFT only | **F: THIS hybrid** |
|---|---|---|---|---|---|---|
| Illumination / sun-angle | ✗ — 17.3 percent SR at lunar pole (CNSFM Table 1) | ~ — day-night evidence, same sensor only (SuperGlue Aachen) | ✗ — not a correspondence method | ✓✓ — 72.3 percent SR at pole vs 31.2 best baseline | ✓ — 100 percent SR on 6 NRD sets vs SIFT 31.7 | ✓ — M2/M1/M3 arbitrated by terrain + confidence, M0 fallback |
| Viewpoint | ~ | ✓ | n/a | ✓ — similarity-invariant crater topology | ✓ — multi-MIM rotation, 100 percent over 360° | ✓ |
| Scale | ✓ | ✓ | ~ | ✓ — diameter/side ratios | ✗ — no scale invariance (closed by our log-Gabor scale-space extension) | ✓ |
| Multi-modal | ✗ | ✗ — unproven cross-sensor | ~ | ~ | ✓ — built for NRD | ✓ |
| Uniform distribution | ~ | ~ | ✗ | ✗ — craters cluster spatially | ~ | ✓✓ — L3 dual-stage selection as an explicit component with its own metric |
| Sub-pixel accuracy | ~ — only with added refinement | ✓ — 0.4–0.9 px reported | ✓ — but grid-level, not correspondence | ~ — 1.0–2.2 px | ~ — 1.8–2.0 px raw | ✓ — L5 refinement forced on every path, gain reported |
| Outlier rejection | ✓ RANSAC | needs DEGENSAC added | n/a | ✓ MCR (RCM 62.2→100 in ablation) | ~ NBCS | ✓✓ — L3 confidence + L4 DEGENSAC + GCP declustering + Z-score |
| Quantified evaluation | ✓ | ✓ | ~ | ✓ | ✓ | ✓✓ — L7 leaderboard incl. spatial coverage + runtime |
| Hackathon feasibility | ✓✓ | ✗ — no lunar training set, no time | ✓ | ~ — detector needs per-sensor retraining | ✓ | ✓ — all components pretrained, classical, or small-train |

Key insight from the research: **no single published method covers all requirements.** Every paper documents a failure regime — SIFT dies at the poles, LoFTR extrapolates out-of-domain, CNSFM dies in mare, RIFT has no scale invariance and is slow, ASP solves geometry but not multi-modal correspondence. Architecture F wins because (1) each failure regime is covered by a different component, (2) the arbiter picks per-pair using measured crater density and confidence, and (3) the L7 leaderboard replaces faith with data. That is the defensible answer to why-this-model.

---

## 11. Stage algorithms (pseudocode + math, implementation-ready)

### 11.1 Coarse localization & bbox padding (L0)

```
pad_m  = k * sigma_pointing          # sigma_pointing ≈ 500–2000 m; k = 2–5
pad_lat_deg = pad_m / 111320
pad_lon_deg = pad_m / (111320 * cos(lat_center))
bbox = footprint_bbox expanded by (pad_lon_deg, pad_lat_deg)
```
Rationale: metadata ephemeris is a search window; padding guarantees the true counterpart is inside the cropped reference patch (SIH26166_reference_mapping.md).

### 11.2 Shadow / validity mask (L1, F5)

```
for each pixel or tile:
    mu, sigma = local stats over window w (e.g. 32 px)
    dark      = mu < tau_b                       # tau_b ≈ 10 (8-bit)
    flat      = sigma < tau_s                    # tau_s ≈ 3 (8-bit)
    cast      = cos(incidence) < tau_i           # tau_i ≈ 0.15, geometry-based
valid_mask = not (dark or flat or cast)
```
All downstream stages (detection, matching, refinement, coverage) ignore masked pixels. Matches whose support patches touch the mask are dropped.

### 11.3 Keypoint ANMS — SSC variant (L3a, F1)

```
input: keypoints with response r, sorted descending
for each keypoint i:
    r_i = distance to nearest keypoint j with r_j > r_i    # radius to a stronger neighbor
# Square Covering: greedily cover the image with squares centered on the
# largest-r points; suppress any point inside an already-covered square
selected = points whose squares are accepted, capped at K
```
Property test (mandatory): variance of per-cell keypoint counts must drop vs top-K baseline. Code base: BAILOOL/ANMS-Codes (SSC).

### 11.4 Coverage-aware match selection (L3b)

```
input: matches m with confidence c(m); grid G of N×N cells; cell cap C; budget K
sort matches by c descending
selected = []
for m in matches:
    cell = cell_of(m.source_xy)
    if count(selected in cell) < C: selected.append(m)
# bisection on tau to hit K exactly (LoFTR-SPP style), with one-to-one fix:
# if two selected matches share a reference keypoint, keep the higher c
coverage = |occupied cells| / |cells intersecting valid_mask|
```

### 11.5 Model ladder (L4, F3)

```
for model in [similarity(4dof), affine(6dof), homography(8dof)]:
    M, inliers = DEGENSAC(matches, model, thr=t_gsd)
    if inlier_ratio >= r_min and RMSE(inliers) <= rmse_ok: accept M; stop
# fallback: tile-wise local models
for each tile T (grid, 50 percent overlap):
    M_T, inliers_T = DEGENSAC(matches in T, affine, thr=t_gsd)
    require len(inliers_T) >= 12; blend M_T with neighbors (weighted by distance)
record chosen model level in RegistrationResult
```
t_gsd scales with reference GSD (0.5–3.0 px); r_min ≈ 0.05; rmse_ok ≈ 1.0 px pre-refinement.

### 11.6 Sub-pixel refinement math (L5, F7)

Normalized cross-correlation over window W around the coarse match:

    NCC(dx,dy) = Σ (I1 - mean1)(I2(dx,dy) - mean2) / (σ1 σ2 (|W|-1))

Phase-only correlation alternative (illumination-tolerant):

    R = (F1 · conj(F2)) / |F1 · conj(F2)|   → peak of IFFT(R) gives integer shift

2D paraboloid sub-pixel peak from integer scores P at (0,0), (±1,0), (0,±1):

    dx = (P[-1,0] - P[+1,0]) / (2 · (P[-1,0] - 2·P[0,0] + P[+1,0]))
    dy = (P[0,-1] - P[0,+1]) / (2 · (P[0,-1] - 2·P[0,0] + P[0,+1]))

Apodization: Gaussian or Tukey window (Blackman measurably worst — HybridPhaseCorrelation). Reject a refinement if peak sharpness P[0,0]/Σ(3×3) < tau_q (unimodal, well-separated peak required). Unit test: synthetic shifted image must recover the shift within 0.05 px.

### 11.7 DEGENSAC parameters (L4)

max 10 000 iterations; confidence 0.99999; reprojection threshold t_gsd per pair; homography mode for orbital pairs; degenerate-config handling ON (large flat maria are quasi-planar); one-to-one constraint applied after.

### 11.8 Per-stage quality gates (accept / reject / fallback)

| Stage | Gate | On failure |
|---|---|---|
| L2 | ≥ 150 candidate matches, ≥ 25 after ratio/confidence | try next matcher in arbitration order |
| L3 | coverage ≥ 0.60 of valid cells, ≥ 25 selected | relax C; then fall back to M0 |
| L4 | inlier_ratio ≥ 0.05, ≥ 20 inliers | widen t_gsd ×1.5 once; then fallback |
| L5 | ≥ 70 percent of inliers refine successfully | keep unrefined positions, flag pair |
| L6 | warp resampling valid over ≥ 90 percent of footprint | report partial registration |

---

## 12. Matcher spec cards (what the benchmark will actually run)

### M0 — SIFT + RANSAC (baseline, always runs)
- Pipeline: tiled SIFT (tile ≈ 1024 px, 50 percent overlap) → BF-L2 k=2 → Lowe ratio 0.75 → RANSAC homography (3 px) per tile → best tiles merged.
- Preprocessing branch: OHRC→NAC gets CLAHE; learned matchers skip heavy branches (F4).
- Literature expectation: works equatorial; fails polar (17.3 percent SR, CNSFM Table 1); SIFT-IIRS-WAC precedent ≈ 73 m RMSE on IIRS↔WAC (sub-pixel vs 80 m GSD).
- Role: floor performance + tie-breaker + the number every other matcher must beat.
- Config keys: nfeatures 4000/tile, ratio 0.75, ransac_thr 3.0, tile 1024.

### M1 — RIFT/RIFT2 + scale-space extension (classical illumination-robust)
- Pipeline: log-Gabor PC (Ns=4, No=6) → corner+edge detection via min/max moment maps → MIM descriptor (6×6×No) → multi-MIM rotation handling (No candidates) → NN match.
- Our extension (novelty claim): multi-octave log-Gabor scale-space so RIFT gains scale invariance for OHRC(0.28 m)↔NAC(0.5 m) / TMC(5 m)↔WAC(100 m) ratios.
- Literature expectation: 100 percent SR across 6 NRD datasets (SIFT 31.7, SAR-SIFT 28.3); raw ME ≈ 1.8 px / RMSE ≈ 1.9 px → L5 refinement pushes toward sub-pixel. Risks: 15–30× SURF runtime; one documented total failure.
- Config keys: Ns 4, No 6, octaves 3, ratio 0.9 (MIM distances are coarser than SIFT).

### M2 — SuperPoint + LightGlue (learned)
- Pipeline: SuperPoint (pretrained, lunar-untrained) → LightGlue (adaptive depth/width, ≈17 ms easy pairs, Apache-2.0) → confidence per match.
- Literature expectation: the only method that succeeded on ALL conditions in the direct benchmark on our sensors (equatorial + polar + SAR-optical, Traditional-vs-DL Tables); RMSE 0.4–0.9 px.
- Mandatory F2 layer: in-domain check, one-to-one, then L4. Domain gap (MegaDepth-style training) is a measured risk, not an assumption.
- Config keys: nms 3 px, kp_limit 2048, lg_thresholds default, fp16 on GPU.

### M3 — Crater-geometry branch (CNSFM-style, gated F14)
- Pipeline: crater detector (YOLOv9 transfer-learned, methodology: 4 682 annotated craters; or DeepMoon / TMC-2 U-Net) → CNSF = central crater + K-NN topology → similarity-invariant parameters (inter-center angles, side-length ratios, diameter ratios) → matching → MCR structural outlier removal (local transform consistency).
- Literature expectation: 100 percent SR equatorial/mid-latitude, 72.3 percent at south pole (baselines ≤ 31.2); RMSE 1.0–2.2 px; RCM 99.3–100.
- Gate: crater density ≥ tau_c (default ≥ 5 craters/Mpx in BOTH images) else skip.
- Config keys: K 6, tau_c 5, detector weights per sensor.

## 13. Dataset engineering (exact workflow)

### 13.1 ISRO product-ID decode

```
ch2_ohr_nrp_20200827T0030107497_d_img_d18
 │   │   │   │                    │   │   │
 │   │   │   │                    │   │   └─ processing/arc version tag
 │   │   │   │                    │   └─ product type: img (raw), oth (ortho), dtm (DEM)
 │   │   │   │                    └─ data level code
 │   │   │   └─ acquisition UTC timestamp
 │   │   └─ mode (nrp/ndn = look/observation mode)
 │   └─ payload: ohr = OHRC, tmc = TMC-2, iir = IIRS
 └─ mission: Chandrayaan-2
```

### 13.2 Stratification plan (50 pairs, leakage-free by geo-cell)

| Stratum | Bins | Pairs |
|---|---|---|
| Latitude | 0–30° / 30–55° / >55° | 16 / 16 / 10 |
| Δ solar azimuth (src vs ref) | <30° / 30–90° / >90° | 10 / 20 / 12 |
| Terrain | highland / maria / crater-sparse young | 25 / 15 / 10 |
| Sensor pair | OHRC→NAC / TMC-2→WAC | 30 / 20 |
(each pair carries all four labels; bins are crossed, not exclusive)

### 13.3 Download workflow

1. Register (manual, one-time): PRADAN pradan.issdc.gov.in + CHMAP chmapbrowse.issdc.gov.in.
2. Pilot set (known-good, from ASP §8.15): ch2_ohr_nrp_20200827T0030107497_d_img_d18 + ch2_ohr_nrp_20200827T0226453039_d_img_d18 + ch2_tmc_ndn_20231101T0125121377_d_oth_d18 (+ _dtm). Region: lon 20–21°, lat −70…−67.
3. Expand: browse CHMAP by stratum bins; download OHRC/TMC zips; keep filenames.
4. References: Lunar ODE (ode.rsl.wustl.edu/moon) product/bbox search for NAC strips; WAC 643 nm global mosaic (18.6 GB, lroc.asu.edu/data) downloaded once, cropped with GDAL by padded bbox; alternative Moon Trek WMTS (trek.nasa.gov/moon).
5. SPICE: downloadIsisData chandrayaan2 $ISISDATA excluding ck; then fetch only the ch2_att_<window>_v1.bc files matching each strip date via rclone.

### 13.4 Ground-truth tie-point protocol

1. Tool: custom matplotlib/QGIS viewer with side-by-side zoom + crosshair.
2. Grid: 6×6 cells over the overlap; target 1 point per cell ⇒ 30–50 points/pair.
3. Placement: crater centers and rim intersections (stable under illumination); log the feature type per point.
4. QC: second annotator re-places 20 percent; disagreement > 1.5 px ⇒ point discarded.
5. Storage: data/metadata/gt/<pair_id>.json with pixel coords + lon/lat + annotator ids.
6. Held-out rule: GT pairs never used for parameter tuning; tuning happens on train cells only (F9).

---

## 14. End-to-end worked example (one OHRC→NAC pair, expected values)

```
Input: ch2_ohr_nrp_20200827T0030107497_d_img_d18 (OHRC, 0.28 m/px, ≈3 km swath)
L0  footprint from XML → bbox ≈ 20.0–21.0 E, −70…−67 N region
    pad: k=3 × sigma≈800 m ⇒ ≈2.4 km → padded bbox
    ODE query → NAC strip M... (0.5 m/px) covering ≈80 percent of footprint
L1  mask: 12 percent of pixels flagged (cast shadow + flat mare patch)
    GSD ratio 0.56 → 1 pyramid level on the NAC side
L2  M2 LightGlue: 4 210 raw matches, confidences in [0.05, 0.99]
L3  conf ≥ 0.5 → 1 800; grid 8×8, cap C=5 → 240 selected; coverage 0.94
L4  DEGENSAC homography, thr 0.8 px → 210 inliers (ratio 0.875)
    RMSE(inliers) = 0.61 px  ⇒ model accepted
L5  NCC 32 px windows, Tukey window, paraboloid ⇒ RMSE 0.34 px
    refinement success 96 percent; median shift correction 0.28 px
L6  registered GeoTIFF + 210 match points (CSV+GCP) + checkerboard QC
L7  RegistrationResult row appended; RMSE 0.34 px, coverage 0.94, runtime 6.1 s
```
If any gate had failed (e.g. inliers < 20), arbitration falls back to M1, then M0, and the fallback is recorded — never silently dropped.

## 15. Full config examples

### configs/ohrc_nac.yaml

```yaml
pair: ohrc_nac
source: {sensor: OHRC, gsd_m: 0.28}
reference: {sensor: NAC, gsd_m: 0.5}
geometry:
  pad_k: 3
  sigma_pointing_m: 800
  ck_glob: ch2_att_*_v1.bc
preprocessing:
  shadow_mask: {window: 32, tau_b: 10, tau_s: 3.0, tau_i: 0.15}
  radiometric: {percentiles: [2, 98], stat_transfer: true}
  sensor_branch: clahe_pca        # learned matchers bypass (F4)
  tiles: {size: 1024, overlap: 0.5}
selection:
  anms: {enabled: true, variant: ssc, budget: 2000}
  grid: {n: 8, cell_cap: 5, budget: 250, min_coverage: 0.60}
registration:
  ransac: {tool: degensac, iters: 10000, confidence: 0.99999}
  model_ladder: [similarity, affine, homography]
  tilewise: {size: 512, overlap: 0.5, min_inliers: 12}
  decluster: {min_spacing_px: 20, zscore: 3.0, min_gcps: 20}
refinement:
  method: ncc            # ncc | poc
  window: 32
  apodization: tukey
  peak_fit: paraboloid
  sharpness_tau: 0.45
evaluation:
  thresholds_px: [1.0, 0.5]
  report: [rmse, medae, inlier_ratio, coverage, runtime]
```

### configs/matchers.yaml

```yaml
registry:
  sift:      {class: matching.sift.SiftMatcher}
  rift2:     {class: matching.rift.RiftMatcher, params: {Ns: 4, No: 6, octaves: 3}}
  lightglue: {class: matching.lightglue.LightGlueMatcher, params: {nms: 3, kp_limit: 2048}}
  crater:    {class: matching.crater.CraterMatcher, params: {K: 6, tau_c: 5}}
arbitration:
  order: [crater, lightglue, rift2, sift]
  fallback_floor: {inlier_ratio: 0.05, min_inliers: 20}
```

---

## 16. Testing & acceptance criteria per layer

| Layer | Test | Acceptance |
|---|---|---|
| L0 | label parser on 5 known products | footprint/solar/UTC fields match XML values exactly; isisimport succeeds without rename |
| L0 | reference query | padded bbox returns a reference patch with overlap ≥ 0.5 for all pilot pairs |
| L1 | shadow mask | masked fraction 5–30 percent on pilot set; no mask on synthetic well-lit pair |
| L1 | normalization | histogram of source after transfer within 10 percent KL of reference |
| L3 | ANMS uniformity | per-cell count variance < top-K baseline variance on every pair |
| L3 | selection | coverage reported before/after; selection is deterministic given seed |
| L4 | synthetic homography | warp a reference crop with known H, add outliers ⇒ recovered H error < 0.1 px, inlier separation 100 percent at 20 percent outlier rate |
| L5 | synthetic sub-pixel | known 0.37 px shift recovered within 0.05 px |
| L7 | leakage audit | script proves zero shared geo-cells between train/test in manifest |
| E2E | smoke pair | full pipeline on pilot pair produces RegistrationResult + GeoTIFF + QC images |

## 17. Compute & storage budget

| Item | Estimate |
|---|---|
| ASP conda env (bundled ISIS/ALE/USGSCSM) | ≈ 10 GB disk |
| CK kernels per strip window | ≈ 1–2 GB total for pilot (never the 200 GB full set) |
| WAC 643 nm mosaic (download once) | 18.6 GB |
| NAC strips (30 pairs) | ≈ 5–15 GB |
| Per-pair runtime: M0 | ≈ 20–60 s CPU |
| Per-pair runtime: M1 RIFT | ≈ 3–10 min CPU (tile-restricted) |
| Per-pair runtime: M2 LightGlue | ≈ 2–6 s GPU (RTX 3050 sufficient), ≈ 30 s CPU |
| L5 refinement | ≈ 1–2 s per 100 matches |
| Total benchmark (4 matchers × 50 pairs) | ≈ 1–2 days single machine, parallelizable per pair |

## 18. SIH deliverables mapping (problem statement → component)

| Problem-statement requirement | Where it is satisfied | Evidence artifact |
|---|---|---|
| Generic correspondence solution for CH-2 optical vs lunar reference | L2 pluggable engine, 3 sensor tracks | leaderboard per sensor pair |
| Sub-pixel accuracy | L5 refinement + L4 model ladder | RMSE / percent<1px / percent<0.5px / MedAE columns |
| Uniform distribution of match points | L3 Uniform Correspondence Optimization (explicit component) | spatial_coverage + grid_density_std, before/after figures |
| Match points product | L6 | match-points CSV + GCP list |
| Registered product | L6 | registered GeoTIFF + checkerboard QC |
| Outlier rejection | L3 confidence + L4 DEGENSAC + declustering + Z-score | inlier_ratio per pair |
| Evaluation metric (RMSE, inlier count, inlier ratio) | L7 harness | results/leaderboard.csv |
| Robust to illumination / viewpoint / scale variation | M1 PC+MIM, M2 learned attention, M3 crater geometry, L1 branches | per-stratum leaderboard rows (latitude × Δ-azimuth × terrain) |

---

**Bottom line:** this is the best architecture available from the research because it is the only one that (a) covers every documented failure regime with a dedicated component, (b) makes the model choice an empirical output rather than an input, (c) treats uniform correspondence selection as a first-class component with its own metric, and (d) is fully implementable inside hackathon constraints with pretrained/classical parts only.

*Next executable step unchanged: P0/P1 — scaffold + label parser + manifest, then manual PRADAN registration and pilot downloads.*
