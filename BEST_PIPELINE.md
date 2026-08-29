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

*Companion docs: BEST_ARCHITECTURE_KNOWN.md (design + evidence). This file (execution). Next executable step: S0/S1 — env setup and ingest on the pilot set.*
