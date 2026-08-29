# SIH26166 — VALIDATION v2.0

How to verify the system is correct. This document defines what "it works" means and how to prove it.

---

## 1. What Must Be Validated

| Item | What "Pass" Means |
|---|---|
| End-to-end pipeline | RMSE < 1.0 px on >=50% of test pairs; no silent failures |
| Matcher M0 (SIFT) | Runs on all pairs; produces baseline metric; fails gracefully on polar |
| Matcher M1 (RIFT2) | SR >= 90% across all terrain classes; RMSE after L5 < 1.0 px |
| Matcher M2 (LightGlue) | Inlier ratio >= 0.20 across diverse pairs; F2 checks never disabled |
| Matcher M3 (Crater) | Only triggers when crater_density >= tau_c; 0% false activation in mare |
| Uniform coverage | grid_density_std <= 4.0; coverage >= 0.60 on all matchers |
| Sub-pixel refinement | refinement_gain >= 0.10 px on >= 60% of pairs (L5 is doing real work) |
| IIRS module | RMSE < 80 m absolute on IIRS pairs vs WAC reference |
| Leakage | No geo_cell overlaps between train and test splits |

---

## 2. Ground Truth Construction

### 2.1 Manual Annotation Protocol

1. Select 15-20 pairs from the test split, stratified by terrain class, latitude bin, and sensor pair
2. For each pair: lay a 6x6 uniform grid over the valid (unmasked) region of the source image
3. For each of the 36 grid points: identify the matching location in the reference image by visual inspection and photoclinometric cues (crater rims, small boulders, texture pattern)
4. Record: src_xy and ref_xy as (col, row) floats; partition as "eval" for held-out, "fit" for validation of matcher consistency
5. Ensure > 20 points in the "eval" partition per pair for RMSE computation
6. Re-annotate 20% of points independently (by a second annotator or after a time gap) for inter-annotator error estimation; record as "qc" partition
7. Store in INTERFACES.md §7 format at data/metadata/gt/<pair_id>_gt.json

### 2.2 Cross-Method Consistency Adjudication

Where manual annotation is too difficult (heavy shadow, low texture):
1. Run all matchers on the pair
2. Identify points where 3 or more matchers agree to within 0.5 px
3. Use consensus as pseudo-GT; label partition="fit" (not "eval")
4. Do not mix pseudo-GT with manual-GT in the same RMSE computation

### 2.3 LOLA/pc_align Anchor (where available)

Where a LOLA track crosses the source footprint:
- Use pc_align (ASP tool) to compute a rigid offset between the pipeline's registered GeoTIFF and the LOLA track
- Report: pc_align residual in meters as an independent absolute accuracy check

---

## 3. Evaluation Dataset Requirements

The test set must include:
- >= 5 pairs from each terrain class: {equatorial_mare, equatorial_highland, polar_highland, polar_mare, crater_floor, ejecta}
- >= 3 pairs from latitude > +/-55 degrees
- >= 3 pairs from delta_azimuth > 90 degrees (extreme illumination change)
- >= 3 pairs from the lowest crater_density bin (tests M3 gating)
- All sensor pair types: OHRC-NAC, OHRC-WAC, TMC-2-WAC, and IIRS-WAC (separate module)

Minimum test set size: 25 pairs across the full stratification.

---

## 4. Metric Definitions

All metrics computed ONLY on the "eval" partition of GT checkpoints.

**RMSE (primary)**
RMSE = sqrt(mean(residuals_squared))
residuals = euclidean distance in pixels between predicted ref_xy and GT ref_xy
Report: before L5 refinement and after; ALWAYS state N (number of GT checkpoints used).

**pct_lt_1px**
Fraction of GT checkpoints with residual < 1.0 px.

**pct_lt_0p5px**
Fraction of GT checkpoints with residual < 0.5 px. The sub-pixel precision indicator.

**MedAE**
Median absolute error in pixels. Robust to outlier GT errors.

**Inlier count / ratio**
inlier_ratio = len(inliers) / len(candidates_after_L3)
inlier_count = absolute count of DEGENSAC inliers

**Spatial coverage**
coverage = occupied_cells / valid_cells
where valid_cells = grid cells with mask_fraction < 0.5

**Grid density std-dev**
std(match_count per cell) over the NxN grid. Lower = more uniform.
Report both before and after L3 selection.

**Refinement gain**
refinement_gain = RMSE_coarse - RMSE_refined (should be positive; negative = refinement hurt)

**Runtime**
Wall-clock time in seconds per pair for S4-S7 (matching through refinement). Reported per matcher.

**Precision, Recall, Matching Score (where GT allows)**
precision = TP / (TP + FP) where TP = predicted match within 3px of GT match
recall = TP / total_GT_matches
matching_score = (precision + recall) / 2

---

## 5. Pass/Fail Criteria (System-Level)

The implemented system passes validation if, on the test split:

| Criterion | Required | Stretch |
|---|---|---|
| Best matcher RMSE (mean across pairs) | < 1.0 px | < 0.5 px |
| Best matcher pct_lt_1px | >= 0.70 | >= 0.85 |
| spatial_coverage (mean) | >= 0.60 | >= 0.75 |
| grid_density_std (mean) | <= 4.0 cells | <= 2.5 cells |
| inlier_ratio (mean) | >= 0.10 | >= 0.25 |
| M0 failure rate (no output) | <= 30% of pairs | <= 15% |
| IIRS RMSE (absolute) | < 80 m | < 40 m |
| Leakage audit | must pass | must pass |
| Polar stratum included in report | mandatory | mandatory |

---

## 6. Leakage Audit Protocol

```bash
python -m src.evaluation.leakage_audit --manifest data/pairs/manifest.jsonl
```

Checks:
- No pair appears in both train and test split
- No pair's geo_cell appears in both splits (the geo_cell is the split unit, not the pair)
- Any gt_path present in manifest must correspond to a pair in the test split
- The leaderboard.csv split column matches manifest.jsonl split for all pair_ids

The leakage audit must pass before any leaderboard number is published or quoted.

---

## 7. Regression Suite

For catching regressions during implementation:

### Unit Tests
- L0: isisimport + spiceinit on a known-good OHRC product
- L0: bbox padding formula (verify math against reference SIFT-IIRS-WAC paper setup)
- L1: shadow mask fraction on one representative pair (expected: 5-30%)
- L1: radiometric normalization (mean/std of normalized src should match ref's after transfer)
- L1: ANMS SSC output has no two points within radius r
- L2: M0 (SIFT) produces >= 50 candidates on a known-good textured pair
- L2: M2 (LightGlue) F2 checks reject out-of-bounds and duplicate matches
- L3: coverage after grid selection is >= coverage_min
- L4: DEGENSAC on a known-good set produces inlier_ratio >= 0.5
- L4: Model ladder selects homography over affine when residuals require it
- L5: refinement gain is positive on a synthetic controlled shift test
- L7: RMSE computation reads only "eval" partition points

### Integration Tests
- Full pipeline on 3 pilot pairs, all matchers, verifies no crashes and all artifacts written
- benchmark.py --resume: re-running does not re-process completed stages

### Synthetic Ground Truth Test
- Take one real image; apply known transform T (rotation=2 deg, scale=1.05, shift=50px each axis)
- Run full pipeline; verify recovered transform is within 0.5 px RMSE of T
- Use this as a daily sanity check before running on real pairs

---

## 8. Known Failure Conditions (Expected, Not Bugs)

| Failure | Cause | Expected? | Action |
|---|---|---|---|
| M0 fails at poles | SIFT gradient collapses near polar terrain | Yes -- documented | M3 or M1 should take over |
| M3 skips in mare | crater_density < tau_c | Yes -- gating works | M2 or M1 runs instead |
| M1 fails on one pair | one documented RIFT total failure mode | Yes -- known | M0 fallback; record in failures.jsonl |
| High mask fraction (>30%) | polar deep shadow | Yes -- a real stratum | Keep pair, proceed on unmasked area |
| tile-wise model chosen | high latitude or high relief | Yes -- expected | Record ladder level; not an error |
| RMSE > 1px for IIRS | 80m GSD + spectral appearance gap | Expected without photometric correction | Apply correction before matching |
| LightGlue domain gap on lunar | MegaDepth training domain | Known risk | F2 checks + M0 fallback mitigate |
