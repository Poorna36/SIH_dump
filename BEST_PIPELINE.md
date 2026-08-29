# SIH26166 — BEST PIPELINE (v1.0)

Executable pipeline runbook for the system defined in `BEST_ARCHITECTURE_KNOWN.md`. The architecture says WHAT each layer is; this file says HOW a pair flows through it: exact stage order, commands, input/output artifacts, success gates, and failure handling.

Convention: every artifact is namespaced by `pair_id` (e.g. `ohr_20200827T0030__nac_M123456789`) and by `matcher` (`sift | rift2 | lightglue | crater`).

---

## 0. Pipeline at a glance

```
 data/raw/ (ISRO zips, original names — F10)
      │ S1 ingest          scripts/ingest.py
      ▼
 data/calibrated/*.cub  +  data/metadata/products.jsonl        [L0]
      │
      │ S2 pair building   scripts/build_pairs.py
      ▼
 data/pairs/manifest.jsonl  (+ reference crops in data/reference/) [L0]
      │
      │ S3 preprocessing   scripts/preprocess.py                 [L1]
      ▼
 data/processed/<pair_id>/  {src_patch, ref_patch, valid_mask, tiles, meta.json}
      │
      │ S4 matching        src/matching (registry loop)          [L2]
      ▼
 results/<pair_id>/<matcher>/matches_raw.json
      │
      │ S5 selection       src/selection (ANMS + grid/coverage)  [L3]
      ▼
 results/<pair_id>/<matcher>/matches_selected.json  (+coverage stats)
      │
      │ S6 verification    src/registration (DEGENSAC + ladder)  [L4]
      ▼
 results/<pair_id>/<matcher>/geometry.json  (model, inliers, residuals)
      │
      │ S7 refinement      src/refinement (NCC/POC + paraboloid) [L5]
      ▼
 results/<pair_id>/<matcher>/matches_refined.json
      │
      │ S8 products        scripts/register.py                   [L6]
      ▼
 results/<pair_id>/<matcher>/  {registered.tif, match_points.csv,
                                match_points.gcp, qc_*.png}
      │
      │ S9 evaluation      src/evaluation                        [L7]
      ▼
 results/leaderboard.csv  +  results/pair_results/*.json  +  arbitration.log
```

## 1. Stage table

| Stage | Script / module | Reads | Writes | Gate to proceed |
|---|---|---|---|---|
| S0 setup | env setup, configs | — | working env + configs | asp env active, ISISDATA set |
| S1 ingest | scripts/ingest.py | data/raw zips | .cub + products.jsonl | spiceinit OK, footprint parsed |
| S2 pairs | scripts/build_pairs.py | products.jsonl, ODE/WMTS | manifest.jsonl + ref crops | overlap ≥ 0.5 recorded |
| S3 preprocess | scripts/preprocess.py | manifest entry | processed/<pair_id>/ | mask fraction 5–30 percent |
| S4 match | src/matching registry | processed patches | matches_raw.json | ≥ 150 candidates |
| S5 select | src/selection | matches_raw.json | matches_selected.json | coverage ≥ 0.60, ≥ 25 matches |
| S6 verify | src/registration | matches_selected.json | geometry.json | inlier_ratio ≥ 0.05, ≥ 20 inliers |
| S7 refine | src/refinement | geometry.json + patches | matches_refined.json | ≥ 70 percent refined |
| S8 products | scripts/register.py | matches_refined.json | tif/csv/gcp/qc | warp valid ≥ 90 percent |
| S9 evaluate | src/evaluation | all of the above | leaderboard.csv | leakage audit passes |

---

## 2. Stage runbook

### S0 — Environment setup (once)

```bash
conda create -n asp -c conda-forge -c usgs-astrogeology ames stereo-pipeline   # ASP ≥ 3.7.0
conda activate asp
export ISISROOT=$CONDA_PREFIX  ISISDATA=$HOME/projects/isisdata  ALESPICEROOT=$ISISDATA
downloadIsisData chandrayaan2 $ISISDATA --exclude="kernels/ck/**"   # non-CK parts
# per-strip CK kernels only (never the 200 GB full set — F10):
rclone --config $ISISROOT/etc/isis/rclone.conf copy chandrayaan2:kernels/ck/ \n  $ISISDATA/chandrayaan2/kernels/ck/ --include="ch2_att_27Jul2020_04Sep2020_v1.bc" -P
```

### S1 — Ingest (L0)

```bash
python scripts/ingest.py --raw data/raw --out data/calibrated \n  --meta data/metadata/products.jsonl --kernels $ISISDATA
```
- Steps per product: unzip in place (never rename — F10) → isisimport → spiceinit/CSM → parse .xml label → record footprint corners, solar incidence/azimuth, UTC, GSD, product_id.
- Outputs: `data/calibrated/<product_id>.cub`; one line per product in `products.jsonl`.
- Success gate: spiceinit exits 0; footprint polygon non-empty; solar angles present.
- On failure: missing CK kernel → fetch the window matching the strip UTC (see S0); isisimport crash → verify original filenames intact.

### S2 — Pair building (L0)

```bash
python scripts/build_pairs.py --products data/metadata/products.jsonl \n  --config configs/ohrc_nac.yaml --ode  # NAC via Lunar ODE; WAC via GDAL crop of local mosaic
python scripts/build_pairs.py --products ... --config configs/tmc_wac.yaml --wac data/reference/wac_643nm.tif
```
- Steps: padded bbox (k × sigma_pointing, F-padded rule) → reference query (ODE bbox / WMTS / GDAL crop) → overlap fraction computed → stratum labels attached (latitude bin, Δ-azimuth bin, terrain class, crater density) → geo-cell assigned → split assigned (F9: disjoint 10°×10° cells).
- Outputs: append `PairRecord` to `data/pairs/manifest.jsonl`; reference crop to `data/reference/crops/<pair_id>.tif`.
- Success gate: overlap_fraction ≥ 0.5 (below that, keep pair but flag `partial_overlap: true`).
- On failure: ODE returns no NAC strip → try WAC crop fallback; still none → skip pair, log reason to `data/pairs/skipped.jsonl`.

### S3 — Preprocessing (L1)

```bash
python scripts/preprocess.py --manifest data/pairs/manifest.jsonl \n  --config configs/ohrc_nac.yaml --out data/processed
```
- Steps per pair: shadow/validity mask (F5: dark + flat + cast rules) → radiometric normalization (2/98 percentile clip, mean/std transfer) → sensor branch (OHRC→NAC: CLAHE+PCA; TMC-2→WAC: histogram-match+CLAHE; learned matchers bypass — F4) → tiling → GSD reconciliation pyramid.
- Outputs under `data/processed/<pair_id>/`: `src.tif`, `ref.tif`, `valid_mask.png`, `tiles.geojson`, `meta.json` (records every transform applied — provenance).
- Success gate: masked fraction within 5–30 percent (outside range → review thresholds, do not silently proceed).

---

### S4 — Matching (L2)

```bash
python scripts/benchmark.py --pair <pair_id> --matcher sift        # single
python scripts/benchmark.py --manifest data/pairs/manifest.jsonl \n  --matchers sift,rift2,lightglue,crater --splits test              # full benchmark
```
- Registry loop (configs/matchers.yaml). Order is arbitration order; M0 always runs regardless.
- Gating before a matcher runs: M3 (crater) requires crater_density ≥ tau_c in BOTH images (F14); M2 requires GPU unless cpu flag set; M1 is tile-restricted to control runtime.
- ANMS (F1a) is applied inside sparse matchers (M0/M1) after detection, before description — so keypoint selection is uniform at the source.
- Outputs: `results/<pair_id>/<matcher>/matches_raw.json` = list of {src_xy, ref_xy, confidence, scale, angle} + runtime + matcher params hash.
- Gate: ≥ 150 candidates. On failure: record empty result; arbitration moves to next matcher.

### S5 — Uniform Correspondence Optimization (L3)

```python
# src/selection/spatial.py — applied to every matcher output (F1b)
selected = confidence_filter(matches, tau)            # tau from matcher card
selected = grid_cap(selected, n=8, cap=5)             # per-cell max-N
selected = coverage_greedy(selected, budget=250)      # bisection on tau
selected = one_to_one(selected)                       # conflict fix
metrics  = {coverage: ..., grid_density_std: ...,     # F13 — reported, not optional
            before_count: ..., after_count: ...}
```
- Outputs: `matches_selected.json` + `selection_stats.json`.
- Gate: coverage ≥ 0.60 and ≥ 25 matches. On failure: relax cap once; then this matcher is marked failed for the pair.

### S6 — Geometric verification (L4)

```python
# src/registration/ladder.py
for model in [similarity, affine, homography]:        # F3 ladder
    M, inliers = degensac(selected, model, thr=t_gsd)  # t_gsd from config
    if inlier_ratio >= 0.05 and rmse(inliers) <= 1.0: break
else:
    M = tilewise_affine(selected, size=512)            # high-relief / >±55° fallback
inliers = decluster(inliers, min_spacing, zscore)      # GCP declustering + Z-filter
inliers = in_domain_and_one_to_one(inliers)           # F2 — mandatory for learned matchers
```
- Outputs: `geometry.json` = model type + parameters + inlier indices + per-inlier residuals + chosen ladder level + t_gsd used.
- Gate: inlier_ratio ≥ 0.05, ≥ 20 inliers. On failure: widen threshold ×1.5 once, retry; then matcher marked failed for the pair.

### S7 — Sub-pixel refinement (L5)

```python
# src/refinement/local.py
for each inlier:
    patch = window(ref, ref_xy, W=32, window=tukey)     # never Blackman (F7)
    (dx, dy, sharp) = ncc_or_poc_peak(src_patch, patch)  # integer peak → paraboloid
    if sharp >= tau_q: refined.append(src_xy + 0, ref_xy + (dx, dy))
```
- Outputs: `matches_refined.json` = refined coordinates + per-match refinement delta + sharpness + success flag; plus before/after RMSE.
- Gate: ≥ 70 percent of inliers refine successfully. On failure: keep unrefined positions for those points and flag the pair `partial_refinement: true`.

---

### S8 — Product generation (L6)

```bash
python scripts/register.py --pair <pair_id> --matcher <matcher> \n  --geometry results/<pair_id>/<matcher>/geometry.json \n  --matches results/<pair_id>/<matcher>/matches_refined.json
```
- Steps: apply model ladder (or tile-wise blend) to warp source onto the reference grid → resample → build match-points CSV + GCP list (pixel coords both sides + lon/lat from reference georeferencing) → render QC artifacts.
- Outputs in `results/<pair_id>/<matcher>/`: `registered.tif` (GeoTIFF on reference grid), `match_points.csv`, `match_points.gcp`, `qc_checkerboard.png`, `qc_matches.png`, `qc_residuals.png`.
- Gate: warp valid over ≥ 90 percent of the footprint. On failure: report partial registration extent in the result JSON.

### S9 — Evaluation (L7)

```bash
python -m src.evaluation.aggregate --results results/ --gt data/metadata/gt/ \n  --out results/leaderboard.csv          # also writes results/pair_results/*.json
python -m src.evaluation.leakage_audit --manifest data/pairs/manifest.jsonl
```
- Metrics per (pair × matcher): RMSE on held-out GT checkpoints, percent<1px, percent<0.5px, MedAE, inlier count, inlier ratio, spatial coverage, grid-density std, refinement gain, runtime, precision/recall where GT allows.
- Aggregation: mean/median per (matcher × sensor-pair × stratum); per-stratum rows are mandatory — polar and >±55° rows are never hidden.
- Arbitration log: which matcher won each pair and why (terrain, confidence, fallback event) → `results/arbitration.log`.
- Leakage audit must pass before any leaderboard number is quoted (F9).

## 3. CLI quick reference

```bash
# full run, one config, all matchers, test split only
python scripts/benchmark.py --config configs/ohrc_nac.yaml --splits test \n  --matchers sift,rift2,lightglue,crater --parallel 4

# resume after interruption (skips pairs whose final artifacts exist)
python scripts/benchmark.py --config configs/ohrc_nac.yaml --resume

# single pair debugging, verbose, keep intermediates
python scripts/benchmark.py --pair <pair_id> --matcher rift2 --keep-intermediate -v

# evaluation only
python -m src.evaluation.aggregate --results results/ --gt data/metadata/gt/
```

## 4. Orchestration rules

1. **Checkpointing**: each stage writes its artifact before the next starts; a stage re-runs only if its output is missing or `--force` is set.
2. **Resume semantics**: benchmark.py treats a (pair, matcher) as done iff `matches_refined.json` + `geometry.json` exist; S8/S9 always re-aggregate.
3. **Parallelism**: pairs are independent — process-pool over pairs; GPU jobs (M2) serialized through a lock.
4. **Provenance**: every artifact JSON carries {config hash, code commit, matcher params hash, timestamps}; products.jsonl and manifest.jsonl are append-only.
5. **Determinism**: seeds fixed for RANSAC and selection; report seed in results for reproducibility.
6. **Never silently drop**: every gate failure is recorded in `results/failures.jsonl` with stage, reason, and the fallback taken.

---

## 5. Failure & retry matrix

| Symptom | Likely cause | Action |
|---|---|---|
| spiceinit fails at S1 | missing/mismatched CK kernel window | fetch ch2_att_*_v1.bc covering the strip UTC; re-run S1 |
| isisimport crashes at S1 | product file was renamed | restore original ISRO filename (F10); re-run |
| ODE returns no NAC strip at S2 | footprint outside NAC coverage | fall back to WAC crop; if still none → skipped.jsonl |
| mask fraction > 30 percent at S3 | extreme shadow scene (polar / low sun) | keep pair (it is a stratum!), flag; matchers proceed on unmasked area |
| candidates < 150 at S4 | textureless mare or GSD mismatch | check GSD pyramid; try next matcher; record failure |
| coverage < 0.60 at S5 | matches clustered (crater cluster) | relax cell cap once; if still low, matcher failed for pair |
| inlier_ratio < 0.05 at S6 | matcher noise or degenerate terrain | widen t_gsd ×1.5 once; else tile-wise fallback; else next matcher |
| tile-wise model chosen | high relief or >±55° latitude | expected, not an error; ladder level recorded in geometry.json |
| refinement success < 70 percent at S7 | multimodal NCC peaks (repetitive craters) | keep unrefined coords for those points; flag pair |
| warp invalid > 10 percent at S8 | model extrapolation at footprint edge | report partial registration; consider tile-wise model |
| GPU OOM (M2) | large patches | reduce kp_limit; or run M2 on CPU (slower); never skip F2 checks |

## 6. Pilot-run checklist (first 3 pairs, end to end)

1. ☐ conda asp env active; ISISDATA/ALESPICEROOT exported; non-CK kernels fetched
2. ☐ CK kernel fetched for strip date 2020-08-27 window
3. ☐ Downloaded from PRADAN/CHMAP: the two verified OHRC strips + TMC-2 ortho/DEM (ASP §8.15 set) — filenames untouched
4. ☐ S1 ingest → 3 products in products.jsonl with footprints + solar angles
5. ☐ S2 build_pairs → ≥ 3 manifest entries; NAC reference crops fetched via ODE; WAC crop fetched locally
6. ☐ S3 preprocess → masks 5–30 percent; meta.json provenance written
7. ☐ S4–S7 for M0 (SIFT) on each pilot pair → geometry.json + matches_refined.json exist
8. ☐ S8 → registered.tif opens in QGIS; checkerboard QC looks aligned by eye
9. ☐ S9 → leaderboard.csv row for M0; leakage audit passes
10. ☐ failures.jsonl reviewed — every gate failure accounted for
11. ☐ Only then: repeat S4–S9 for rift2, lightglue (and crater if density gate passes)

## 7. Definition of done (pipeline)

- All 10 stages runnable end-to-end from the CLI on a fresh checkout with only config changes.
- Every artifact in §0 produced for at least 3 pilot pairs × M0.
- leaderboard.csv + arbitration.log + failures.jsonl generated and leakage audit green.
- No stage advances past a failed gate without a recorded fallback.
- Rerunning with the same seed reproduces byte-identical match sets (determinism check).

---


---

# PIPELINE DEEP-DIVE ADDENDUM (v1.1)

## 8. Stage dependency graph, state machine, and full file inventory

### 8.1 Stage dependencies

```
S0 (env, once)
 └▶ S1 (per product)   ── network: SPICE CK kernels only
     └▶ S2 (per pair)   ── network: ODE / Moon Trek / WAC crop
         └▶ S3 (per pair)             # shared by ALL matchers of the pair
             └▶ S4 (per pair × matcher)
                 └▶ S5 ─▶ S6 ─▶ S7    # per (pair, matcher), restartable
                     └▶ S8 (per pair × matcher, needs S7 of that matcher)
                         └▶ S9 (global aggregation, needs S8 of all)
```
Key rule: S3 runs ONCE per pair; every matcher consumes the same preprocessed patches — this is what keeps the benchmark fair.

### 8.2 Pair×matcher state machine

```
EMPTY → MATCHED → SELECTED → VERIFIED → REFINED → REGISTERED → EVALUATED
   └──────────── any state ──▶ FAILED(reason)  # recorded in failures.jsonl;
                                              # arbitration moves to next matcher
```
Resume rule (benchmark.py --resume): the state is the deepest artifact present on disk.

| Deepest artifact present | Resumed state | Next action |
|---|---|---|
| none | EMPTY | S4 |
| matches_raw.json | MATCHED | S5 |
| + matches_selected.json | SELECTED | S6 |
| + geometry.json | VERIFIED | S7 |
| + matches_refined.json | REFINED | S8 |
| + registered.tif | REGISTERED | S9 only |
| + pair_results JSON | EVALUATED | skip |

### 8.3 Full file inventory (everything the pipeline can write)

```
data/
  raw/…                                        # ISRO zip contents, untouched names (F10)
  calibrated/<product_id>.cub
  reference/wac_643nm.tif
  reference/crops/<pair_id>.tif
  pairs/manifest.jsonl        pairs/skipped.jsonl
  processed/<pair_id>/{src.tif, ref.tif, valid_mask.png, tiles.geojson, meta.json}
  metadata/products.jsonl     metadata/gt/<pair_id>.json     metadata/kernels.log
results/
  <pair_id>/<matcher>/matches_raw.json
  <pair_id>/<matcher>/matches_selected.json   <pair_id>/<matcher>/selection_stats.json
  <pair_id>/<matcher>/geometry.json
  <pair_id>/<matcher>/matches_refined.json
  <pair_id>/<matcher>/{registered.tif, match_points.csv, match_points.gcp,
                       qc_checkerboard.png, qc_matches.png, qc_residuals.png}
  pair_results/<pair_id>.<matcher>.json
  leaderboard.csv   arbitration.log   failures.jsonl   run.log
```

---

## 9. Artifact schemas (field by field, implementation contract)

### 9.1 metadata/products.jsonl (one JSON object per product)

```json
{
  "product_id": "ch2_ohr_nrp_20200827T0030107497_d_img_d18",
  "sensor": "OHRC",
  "path_raw": "data/raw/ch2_ohr_…/", "path_cub": "data/calibrated/ch2_ohr_….cub",
  "utc": "2020-08-27T00:30:10.749700", "gsd_m": 0.28,
  "footprint_lonlat": [[20.01,-70.0],[21.0,-70.0],[21.0,-67.0],[20.01,-67.0]],
  "solar": {"incidence_deg": 54.3, "azimuth_deg": 192.1},
  "kernel_files": ["ch2_att_27Jul2020_04Sep2020_v1.bc"],
  "provenance": {"config_hash": "a1b2c3", "commit": "…", "ts": "…"}
}
```

### 9.2 results/<pair>/<matcher>/matches_raw.json

```json
{
  "pair_id": "…", "matcher": "lightglue", "params_hash": "sha1…",
  "seed": 7, "runtime_s": 4.2,
  "kp_src_count": 2048, "kp_ref_count": 1980, "n_candidates": 4210,
  "matches": [
    {"i": 0, "src": [512.4, 88.2], "ref": [301.1, 44.9],
     "conf": 0.93, "scale": 1.0, "angle_deg": 12.5, "tile": 17}
  ]
}
```
Field notes: `conf` ∈ [0,1] — learned matchers emit native confidence; sparse matchers without scores emit `conf = 1 − ratio_distance_normalized`; `tile` = −1 when untiled.

### 9.3 results/<pair>/<matcher>/selection_stats.json

```json
{
  "tau": 0.50, "grid_n": 8, "cell_cap": 5, "budget": 250,
  "before_count": 1800, "after_count": 240,
  "cells_total": 58, "cells_occupied": 55, "coverage": 0.948,
  "grid_density_std_before": 12.4, "grid_density_std_after": 0.9
}
```
`cells_total` counts only cells intersecting the valid mask (F5) — coverage is never diluted by impossible cells.

### 9.4 results/<pair>/<matcher>/geometry.json

```json
{
  "model": "homography", "level": 3, "t_gsd_px": 0.8,
  "params": [[1.0, 0.0, 0.0], [0.0, 1.0, 0.0], [0.0, 0.0, 1.0]],
  "tile_params": null,
  "degensac": {"iters": 10000, "confidence": 0.99999, "seed": 7},
  "n_inliers": 210, "inlier_ratio": 0.875, "rmse_px": 0.61,
  "inlier_indices": [0, 5, 9], "residuals_px": [0.11, 0.07, 0.19],
  "decluster": {"min_spacing_px": 20, "zscore": 3.0, "removed": 6},
  "f2_checks": {"in_domain": true, "one_to_one": true}
}
```
`level`: 1=similarity, 2=affine, 3=homography, 4=tilewise_affine. When `model` is `tilewise_affine`, `params` is null and `tile_params` holds a list of {tile_id, affine_2x3, center} entries.

### 9.5 results/<pair>/<matcher>/matches_refined.json

```json
{
  "n_refined": 210, "n_success": 202, "success_rate": 0.962,
  "rmse_before_px": 0.61, "rmse_after_px": 0.34,
  "refined": [
    {"i": 0, "src": [512.4, 88.2], "ref_coarse": [301.1, 44.9],
     "ref_refined": [301.28, 45.03], "delta": [0.18, 0.13],
     "sharpness": 0.71, "success": true}
  ]
}
```

### 9.6 results/failures.jsonl (append-only audit)

```json
{"ts": "…", "pair_id": "…", "matcher": "rift2", "stage": "S6",
 "gate": "inlier_ratio", "value": 0.031, "threshold": 0.05,
 "action": "widened_t_then_failed", "next_matcher": "sift"}
```

---

## 10. S4 internals — per-matcher implementation procedures

### 10.1 M0 SIFT (sift.py)

```
1. tile loop: 1024 px, stride 512; skip tiles with < 256 px of valid mask
2. per tile: SIFT(nfeatures=4000, contrastThreshold=0.04, edgeThreshold=10)
   keypoints falling on masked pixels are dropped (valid_mask lookup)
3. F1a: ANMS-SSC per image, budget 2000 (on the union of tile keypoints)
4. describe → BF knn k=2 (L2) → Lowe ratio 0.75 → per-tile candidate list
5. map tile-local coords to global; dedupe matches within 1 px
6. emit matches_raw.json; target runtime 20–60 s per pair
```

### 10.2 M1 RIFT/RIFT2 + scale-space (rift.py)

```
1. log-Gabor bank: Ns=4 scales, No=6 orientations; our extension: convolve at
   3 pyramid octaves (scale-space addition — the novelty claim)
2. PC maps from filter-response moments; corners = min-moment map maxima,
   edges = max-moment map + FAST
3. MIM descriptor: per-pixel argmax orientation index; 6×6 spatial × No bins
4. ANMS-SSC budget 2000 (F1a); match: L1 on MIM, ratio 0.9
5. rotation invariance: No rotated-MIM candidates per reference keypoint
6. our scale-consistency filter: reject pair when
   | log(scale_src/scale_ref) − log(gsd_ratio) | > 0.3
7. tile restriction: only tiles with valid fraction > 0.3 (runtime control)
```

### 10.3 M2 SuperPoint + LightGlue (lightglue.py)

```
1. match at reconciled GSD (pyramid level from S3); record the level
2. SuperPoint extract: kp_limit 2048, nms 3 px, on masked images
3. LightGlue match: default filter thresholds; confidences per match
4. map coordinates back to native resolution (scale factor from meta.json)
5. F2 (mandatory): drop matches with either endpoint within 2 px of the image
   edge or on masked pixels; enforce one-to-one by highest confidence
6. fp16 on GPU; CPU fallback ~30 s per pair (never skips F2)
```

### 10.4 M3 crater-geometry (crater.py)

```
1. detect craters on both images (grayscale, resized to 640 px):
   YOLOv9 transfer-learned weights per sensor; DeepMoon as fallback detector
2. craters = (center_x, center_y, radius_px); radii converted to meters via GSD
3. CNSF build: for each crater, K=6 nearest neighbors; features per CNSF =
   inter-center angles, neighbor-distance ratios, diameter ratios
   (all similarity-invariant → illumination-independent by construction)
4. gate (F14): proceed only if crater_density ≥ tau_c (5 /Mpx) in BOTH images
5. candidate CNSF pairs: nearest feature-space neighbors, ratio 0.85
6. cluster consistency: two CNSFs agree if their implied scale and rotation
   agree within tolerance; keep the largest consistent cluster
7. MCR outlier removal: local similarity from crater triples; drop craters
   inconsistent with the local consensus
8. emit crater-center matches, conf = cluster support normalized
```

---

## 11. S5 / S6 internals — algorithms with acceptance math

### 11.1 ANMS-SSC implementation (selection/anms.py)

```
input: keypoints sorted by response desc (n up to ~20k)
1. r_i = distance from point i to its nearest STRONGER point (KD-tree,
   inserted incrementally so only stronger points are queried)
2. square covering: iterate points in radius-descending order; maintain the
   set of accepted squares (center, half_side = r_i/√2); accept point i iff
   its center lies outside all accepted squares; stop at budget K
output: K keypoints with maximal spread + local strength
complexity: O(n log n + K·n) — fine for 20k keypoints
```
Test hook: per-cell count variance of the selected set must be < top-K baseline variance (assert in CI).

### 11.2 Coverage-aware selection with tau bisection (selection/spatial.py)

```
def select(matches, tau):
    m = [x for x in matches if x.conf >= tau]                # confidence filter
    m = grid_cap(m, n=8, cap=5)                              # per-cell max-N
    m = greedy_coverage(m, budget=250)                       # fill sparse cells first
    return one_to_one(m)                                     # highest conf wins

lo, hi = 0.0, 1.0
for _ in range(20):                                          # bisection on tau
    mid = (lo + hi) / 2
    if len(select(matches, mid)) >= budget: lo = mid
    else: hi = mid
final = select(matches, lo)                                  # deterministic tie-break:
                                                             # conf desc, then index asc
```
Report coverage BEFORE and AFTER (F13) — the delta is a headline result figure.

### 11.3 Model ladder acceptance (registration/ladder.py)

```
accept(model) iff:
    n_inliers  >= 20
    inlier_ratio >= r_min = 0.05
    rmse_inliers_px <= 1.0        # pre-refinement ceiling; L5 pushes below
order: similarity (4 dof) → affine (6) → homography (8) → tilewise (fallback)
tilewise: grid 512 px, 50 percent overlap; per tile affine via DEGENSAC
    needs >= 12 inliers; tiles failing use the nearest passing tile model
    blend: M(x) = Σ w_T(x)·M_T(x) / Σ w_T(x),  w_T(x) = exp(−‖x−c_T‖²/(2σ²)), σ = 256
```

### 11.4 GCP declustering + Z-filter (registration/decluster.py)

```
1. sort inliers by residual ascending
2. walk a grid of cell = min_spacing_px (20, GSD-scaled); keep the first
   (best-residual) inlier per cell; discard the rest
3. Z-filter (only if n >= 20): mean μ, std σ of residuals; drop |r − μ| > 3σ;
   repeat once (μ, σ recomputed) — from SIFT-IIRS-WAC, verbatim
4. removed counts land in geometry.json.decluster
```

## 12. S7 internals — sub-pixel refinement (refinement/local.py)

```
per inlier:
1. cut W×W window (W=32) in ref at ref_coarse, and search window ±8 px in src
2. apodize both with a Tukey window (never Blackman — F7)
3. NCC surface S(dx,dy) over the search grid (or POC: cross-power spectrum of
   64×64 windows → IFFT peak; POC preferred when radiometry still differs)
4. integer peak (x0,y0); paraboloid sub-pixel correction on the 3×3 surface:
      dx = (S[x0−1,y0] − S[x0+1,y0]) / (2·(S[x0−1,y0] − 2·S[x0,y0] + S[x0+1,y0]))
      dy analogous on the y axis
5. sharpness = S[x0,y0] / Σ(3×3 surface); reject if < tau_q = 0.45
6. reject also if: window variance < tau_v (flat), OR a second peak > 0.8·peak
   exists in the window (repetitive crater texture — multimodal risk)
7. large GSD-ratio pairs: refine per pyramid level, accumulate offsets
   coarse→fine (HybridPhaseCorrelation pattern)
```
Gate: success_rate ≥ 0.70; per-point failures keep coarse coords and set success=false (never dropped silently).

---

## 13. S8 internals — warping and product formats

### 13.1 Warping

```
src → ref mapping from geometry.json:
  level 1–3: cv2.warpPerspective / warpAffine with the fitted params
  tilewise : per-tile affine maps evaluated on a coordinate grid, blended with
             the Gaussian weights of §11.3, then resampled
resampling: cubic for the image, nearest for the mask
output grid = reference grid exactly (same transform + CRS, e.g. IAU_2015:30100)
valid-fraction check: fraction of output pixels with resample data ≥ 0.90
```

### 13.2 Match-point products

match_points.csv (one row per refined inlier):

```
pair_id, matcher, idx, src_x, src_y, ref_x, ref_y, lon_deg, lat_deg,
residual_px, confidence, sharpness
```

match_points.gcp (GDAL GCP format, one line per point):

```
map_x map_y pixel line lon lat
```
where pixel/line are SOURCE coordinates and map_x/map_y/lon/lat come from the
reference georeferencing — directly ingestible by gdal_translate -gcp.

### 13.3 QC artifacts

```
qc_checkerboard.png : 128 px alternating tiles of warped-source and reference
qc_matches.png      : side-by-side with selected matches drawn + lines
qc_residuals.png    : Gaussian-splatted residual heat map over the footprint
```

## 14. S9 internals — metric definitions and leaderboard spec

### 14.1 Metric formulas (computed on HELD-OUT GT checkpoints only)

```
error_i        = ‖ M(src_gt_i) − ref_gt_i ‖          # model maps src → ref
RMSE           = sqrt( mean_i( error_i² ) )
MedAE          = median_i( error_i )
pct_lt_1px     = 100 · count(error_i < 1.0) / N
pct_lt_0p5px   = 100 · count(error_i < 0.5) / N
inlier_ratio   = n_inliers / n_selected
spatial_coverage = cells_occupied / cells_total_valid
refinement_gain  = (rmse_before − rmse_after) / rmse_before
precision      = correct_matches / selected_matches    # correct = error < 1.0 px
recall         = correct_matches / GT_points_in_overlap
```
IIRS track: additionally report RMSE_meters (px × 80 m).

### 14.2 leaderboard.csv columns (fixed order)

```
matcher, sensor_pair, stratum_latitude, stratum_dazimuth, stratum_terrain,
n_pairs, rmse_px_mean, rmse_px_median, medae_px, pct_lt_1px, pct_lt_0p5px,
inlier_count_mean, inlier_ratio_mean, coverage_mean, refinement_gain_mean,
runtime_s_mean, precision, recall, wins, ties, losses
```
Aggregation rules: one row per (matcher × sensor_pair × stratum); global row
per matcher at the end; wins/ties/losses = paired comparison against M0 on
the common pair subset (a pair where either matcher failed counts as a loss
for the failing matcher — failures are never hidden).

---

## 15. IIRS side-track pipeline deltas (F12)

The IIRS track reuses the S0–S9 skeleton with these substitutions; everything
runs via `--track iirs` using configs/iirs_wac.yaml.

| Stage | Delta from the main track |
|---|---|
| S1i | QUB ingest per ISIS3 IIRS documentation; radiometric + photometric correction (incidence/emission/phase); band aggregation to a grayscale proxy (mean reflectance over a selectable µm range) |
| S2i | reference = WAC crop (100 m/px); GSD ratio ≈ 1.25; footprint from QUB metadata |
| S3i | photometric normalization replaces CLAHE branch; no learned matchers (F4/F12) |
| S4i | M0 SIFT only in v1; M1 as stretch goal; M2/M3 disabled |
| S7i | refinement window scaled to GSD (W=32 at 80 m/px covers 2.56 km) |
| S9i | accuracy target = sub-pixel relative to 80 m GSD; report RMSE_px AND RMSE_meters; benchmark precedent ≈ 73 m (SIFT-IIRS-WAC) |
Failure expectation: optical-vs-hyperspectral is a large modality gap — partial failure is an accepted, reported outcome, not a pipeline bug.

## 16. Test matrix, logging, and exit codes

### 16.1 Test matrix (CI)

| Test id | Stage | Type | Input | Assertion |
|---|---|---|---|---|
| T01 | S1 | unit | 5 known labels | footprint/solar/UTC fields exact |
| T02 | S2 | unit | fixture footprints | padding math matches §11 formulas ±1e-6 |
| T03 | S3 | unit | synthetic image | mask flags only intended pixels |
| T04 | S4 | unit | synthetic pair | SIFT ≥ 50 matches on textured fixture |
| T05 | S5 | unit | synthetic clustered matches | ANMS variance < top-K variance; budget hit ±1 |
| T06 | S6 | integration | known H + 20 percent outliers | H recovered < 0.1 px; inliers exact |
| T07 | S7 | unit | 0.37 px shifted patch | recovered shift within 0.05 px |
| T08 | S8 | integration | synthetic homography | warp residual < 0.05 px at corners |
| T09 | S9 | unit | crafted results | metric formulas match hand-computed values |
| T10 | global | audit | any manifest | zero shared geo-cells across splits |
| T11 | E2E | smoke | pilot pair | full artifact set of §8.3 exists |

### 16.2 Logging

- `results/run.log`: human-readable, one line per stage start/end with duration.
- Every artifact JSON carries provenance {config_hash, commit, seed, ts}.
- failures.jsonl (§9.6) is the single source of truth for gate failures.

### 16.3 Exit codes

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | configuration error (bad yaml / unknown matcher) |
| 2 | data missing (no manifest entry / artifact absent on resume) |
| 3 | all matchers failed gates for a pair (recorded, run continues) |
| 4 | network error on ODE/WMTS/PRADAN fetch (retryable, backoff ×2, max 5) |

---

*Companion docs: BEST_ARCHITECTURE_KNOWN.md (design + evidence) · BEST_PIPELINE.md (this file: execution). Next executable step unchanged: S0/S1 — env setup and ingest on the pilot set.*
