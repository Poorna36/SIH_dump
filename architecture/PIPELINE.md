# SIH26166 — PIPELINE (v2.0)

Executable pipeline runbook for the system defined in `ARCHITECTURE.md`. The architecture says
WHAT each layer is; this file says HOW a pair flows through it: exact stage order, commands,
input/output artifacts, success gates, and failure handling.

**v2.0 change vs v1.0 (BEST_PIPELINE.md):** Stage **S4.5 — MSM Selection** is inserted between
S3 (Preprocessing) and S4 (Matching). All other stages (S0–S9) are unchanged except for the
minor extension of S9 to include `--mode msm_eval`. The existing S4 stage table entry for M3
gating is unchanged; MSM simply adds an earlier, ML-driven routing layer on top of those gates.

Convention: every artifact is namespaced by `pair_id` and by `matcher` (`sift|rift2|lightglue|crater`).

---

## 0. Pipeline at a glance (v2.0)

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
       │ S3 preprocessing   scripts/preprocess.py               [L1]
       ▼
 data/processed/<pair_id>/  {src_patch, ref_patch, valid_mask, tiles, meta.json}
       │
       │ S4.5 MSM selection src/selector/model.py               [L1.5] ← NEW
       ▼
 results/<pair_id>/selector.json  +  msm_features.json          [NEW]
       │ (routing: selected_matcher, confidence, fallback, reason)
       │
       │ S4 matching        src/matching (registry; routed)      [L2]
       ▼
 results/<pair_id>/<matcher>/matches_raw.json
       │
       │ S5 selection       src/selection (ANMS + grid/coverage) [L3]
       ▼
 results/<pair_id>/<matcher>/matches_selected.json  (+coverage stats)
       │
       │ S6 verification    src/registration (DEGENSAC + ladder) [L4]
       ▼
 results/<pair_id>/<matcher>/geometry.json  (model, inliers, residuals)
       │
       │ S7 refinement      src/refinement (NCC/POC + paraboloid)[L5]
       ▼
 results/<pair_id>/<matcher>/matches_refined.json
       │
       │ S8 products        scripts/register.py                  [L6]
       ▼
 results/<pair_id>/<matcher>/  {registered.tif, match_points.csv,
                                match_points.gcp, qc_*.png}
       │
       │ S9 evaluation      src/evaluation                       [L7]
       ▼
 results/leaderboard.csv  +  results/pair_results/*.json
 results/msm_benchmark.csv  +  results/msm_fallback.jsonl       [NEW when --mode msm_eval]
 arbitration.log
```

---

## 1. Stage table (v2.0)

| Stage | Script / module | Reads | Writes | Gate to proceed |
|---|---|---|---|---|
| S0 setup | env setup, configs | — | working env + configs | asp env active, ISISDATA set |
| S1 ingest | scripts/ingest.py | data/raw zips | .cub + products.jsonl | spiceinit OK, footprint parsed |
| S2 pairs | scripts/build_pairs.py | products.jsonl, ODE/WMTS | manifest.jsonl + ref crops | overlap ≥ 0.5 recorded |
| S3 preprocess | scripts/preprocess.py | manifest entry | processed/<pair_id>/ | mask fraction 5–30% |
| **S4.5 MSM** | **src/selector/model.py** | **PairRecord + meta.json** | **selector.json + msm_features.json** | **SelectorResult valid; routing_reason set** |
| S4 match | src/matching registry | processed patches; routing from S4.5 | matches_raw.json | ≥ 150 candidates |
| S5 select | src/selection | matches_raw.json | matches_selected.json | coverage ≥ 0.60, ≥ 25 matches |
| S6 verify | src/registration | matches_selected.json | geometry.json | inlier_ratio ≥ 0.05, ≥ 20 inliers |
| S7 refine | src/refinement | geometry.json + patches | matches_refined.json | ≥ 70% refined |
| S8 products | scripts/register.py | matches_refined.json | tif/csv/gcp/qc | warp valid ≥ 90% |
| S9 evaluate | src/evaluation | all of the above | leaderboard.csv [+ msm_benchmark.csv] | leakage audit passes |

---

## 2. Stage runbook

### S0 — Environment setup (once — unchanged)

```bash
conda create -n asp -c conda-forge -c usgs-astrogeology ames stereo-pipeline  # ASP ≥ 3.7.0
conda activate asp
export ISISROOT=$CONDA_PREFIX  ISISDATA=$HOME/projects/isisdata  ALESPICEROOT=$ISISDATA
downloadIsisData chandrayaan2 $ISISDATA --exclude="kernels/ck/**"
rclone --config $ISISROOT/etc/isis/rclone.conf copy chandrayaan2:kernels/ck/ \
  $ISISDATA/chandrayaan2/kernels/ck/ --include="ch2_att_27Jul2020_04Sep2020_v1.bc" -P

# Install LightGBM for MSM (inside the same conda env):
pip install lightgbm scikit-learn
```

### S1 — Ingest (L0 — unchanged)

```bash
python scripts/ingest.py --raw data/raw --out data/calibrated \
  --meta data/metadata/products.jsonl --kernels $ISISDATA
```

### S2 — Pair building (L0 — unchanged except `msm_split` field added)

```bash
python scripts/build_pairs.py --products data/metadata/products.jsonl \
  --config configs/ohrc_nac.yaml --ode
python scripts/build_pairs.py --products ... --config configs/tmc_wac.yaml \
  --wac data/reference/wac_643nm.tif
```
- `msm_split` defaults to the same value as `split`. Override individually with
  `--msm-split-override <pair_id>=test` if needed for a secondary MSM hold-out.
- All other behaviour unchanged.

---

### S3 — Preprocessing (L1 — extended to write MSM features into meta.json)

```bash
python scripts/preprocess.py --manifest data/pairs/manifest.jsonl \
  --config configs/ohrc_nac.yaml --out data/processed
```

**New fields written to `data/processed/<pair_id>/meta.json`** (see FEATURES.md for definitions):

```json
{
  "masked_fraction":       0.12,
  "src_texture_contrast":  18.4,
  "ref_texture_contrast":  14.1,
  "src_mean_gradient":     9.7,
  "ref_mean_gradient":     7.2,
  "tile_count":            12
}
```

These fields are computed cheaply as by-products of the existing shadow mask and normalization
steps. No additional image I/O is required.

- Success gate: masked fraction within 5–30%. Outside range → review, do not silently proceed.

---

### S4.5 — MSM Selection (L1.5 — NEW)

```bash
python -m src.selector.model \
  --pair <pair_id> \
  --manifest data/pairs/manifest.jsonl \
  --processed data/processed \
  --config configs/msm.yaml \
  --out results/<pair_id>/selector.json
```

Or, when called via `scripts/benchmark.py`, this stage runs automatically between S3 and S4.

**Internal steps:**

```python
# src/selector/model.py (abbreviated)
fv      = extract_features(pair_record, l1_meta)   # 13 features (see FEATURES.md)
probs   = model.predict_proba([fv.to_array()])[0]  # LightGBM; shape (4,)
probs   = apply_hard_rules(probs, pair_record, cfg) # zero + renormalize
sel_idx = np.argmax(probs)
conf    = probs[sel_idx]
fb_idx  = np.argsort(probs)[-2]

if conf >= cfg.tau_high:          routing = "high_confidence"
elif conf >= cfg.tau_low:         routing = "low_confidence_fallback"
else:                             routing = "very_low_confidence_full_mode"
```

**Outputs:**
- `results/<pair_id>/selector.json` — SelectorResult (see ARCHITECTURE.md §4).
- `results/<pair_id>/msm_features.json` — raw FeatureVector (for audit / retraining).

**Mode behaviour:**

| `msm.enabled` config | Behaviour |
|---|---|
| `false` (benchmark / training phase) | Stage writes selector.json with `routing_reason: benchmark_mode`; all matchers run in S4 |
| `true` (production) | S4 runs only the matcher(s) indicated by `routing_reason` |

**Gate:** SelectorResult must be a valid JSON object with all required fields. On failure (model
file missing, feature extraction error): log warning to `results/failures.jsonl` and fall back
to `benchmark_mode` (all matchers run). **The MSM never blocks the pipeline.**

---

### S4 — Matching (L2 — unchanged except MSM routing is respected)

```bash
# Benchmark / training mode (MSM disabled):
python scripts/benchmark.py --pair <pair_id> --matchers sift,rift2,lightglue,crater

# MSM production mode (MSM enabled):
python scripts/benchmark.py --pair <pair_id> --use-selector results/<pair_id>/selector.json
```

In MSM production mode, `benchmark.py` reads `selector.json` and passes only the routed
matcher(s) to the registry loop. All existing gating rules still apply (M3 crater gate, M2 GPU
gate) — the MSM hard-rule masking in S4.5 means M3/M2 are never routed to a pair that would
fail the gate anyway, but the gate check in S4 is kept as a defensive layer.

- Gate: ≥ 150 candidates. On failure: if MSM had routed to a non-M0 matcher, automatically try
  M0 as the safe fallback and record in `results/msm_fallback.jsonl`.

```json
// results/msm_fallback.jsonl (one JSON per event)
{
  "pair_id": "...",
  "event": "s4_gate_failure",
  "routed_matcher": "rift2",
  "fallback_matcher": "sift",
  "gate_value": 87,
  "gate_threshold": 150,
  "timestamp": "2026-08-31T12:00:00Z"
}
```

### S5 — Uniform Correspondence Optimization (L3 — unchanged)

```python
# src/selection/spatial.py
selected = confidence_filter(matches, tau)
selected = grid_cap(selected, n=8, cap=5)
selected = coverage_greedy(selected, budget=250)
selected = one_to_one(selected)
metrics  = {coverage: ..., grid_density_std: ..., before_count: ..., after_count: ...}
```

Gate: coverage ≥ 0.60 and ≥ 25 matches. On failure: relax cap once; then record this matcher as
failed for the pair. In MSM production mode, automatically escalate to M0 fallback if the routed
matcher fails here too.

### S6 — Geometric verification (L4 — unchanged)

```python
for model in [similarity, affine, homography]:
    M, inliers = degensac(selected, model, thr=t_gsd)
    if inlier_ratio >= 0.05 and rmse(inliers) <= 1.0: break
else:
    M = tilewise_affine(selected, size=512)
inliers = decluster(inliers, min_spacing, zscore)
inliers = in_domain_and_one_to_one(inliers)
```

Gate: inlier_ratio ≥ 0.05, ≥ 20 inliers. Widen t_gsd ×1.5 once; then fallback.

### S7 — Sub-pixel refinement (L5 — unchanged)
### S8 — Product generation (L6 — unchanged)

### S9 — Evaluation (L7 — extended for MSM)

```bash
# Standard leaderboard (unchanged):
python -m src.evaluation.aggregate --results results/ --gt data/metadata/gt/ \
  --out results/leaderboard.csv
python -m src.evaluation.leakage_audit --manifest data/pairs/manifest.jsonl

# MSM benchmark (new; run after P5 and after MSM is trained):
python scripts/benchmark.py --config configs/ohrc_nac.yaml \
  --splits test --mode msm_eval \
  --selector models/msm_v1.pkl \
  --out results/msm_benchmark.csv

# Extended leakage audit (checks MSM train/test cell disjointness):
python -m src.evaluation.leakage_audit --manifest data/pairs/manifest.jsonl --check-msm
```

**MSM benchmark outputs** (`results/msm_benchmark.csv`):

| Column | Description |
|---|---|
| `pair_id` | Pair identifier |
| `msm_selected` | Matcher chosen by MSM |
| `msm_confidence` | Selector confidence |
| `msm_routing_reason` | `high_confidence` / `low_confidence_fallback` / `very_low_confidence_full_mode` |
| `oracle_best` | True best matcher (from all-matchers leaderboard on this pair) |
| `msm_correct` | 1 if `msm_selected == oracle_best` |
| `msm_rmse_px` | RMSE of MSM-selected result |
| `oracle_rmse_px` | RMSE of oracle-best result |
| `rmse_degradation_px` | `msm_rmse_px - oracle_rmse_px` (positive = worse) |
| `msm_runtime_s` | Runtime of MSM-selected path |
| `all_matchers_runtime_s` | Runtime of all-matchers path |
| `runtime_reduction_pct` | `(1 - msm_runtime_s/all_matchers_runtime_s) * 100` |

---

## 3. CLI quick reference (v2.0)

```bash
# Full benchmark run, all matchers (MSM disabled — training data collection):
python scripts/benchmark.py --config configs/ohrc_nac.yaml --splits test \
  --matchers sift,rift2,lightglue,crater --parallel 4

# Train MSM after benchmark (requires ≥ 50 labeled train pairs):
python scripts/train_msm.py \
  --leaderboard results/leaderboard.csv \
  --manifest data/pairs/manifest.jsonl \
  --processed data/processed \
  --split msm_split --split-value train \
  --out models/msm_v1.pkl \
  --config configs/msm.yaml

# MSM benchmark on test split (validates MSM before production activation):
python scripts/benchmark.py --config configs/ohrc_nac.yaml --splits test \
  --mode msm_eval --selector models/msm_v1.pkl --out results/msm_benchmark.csv

# Production run (MSM enabled — only after msm_eval passes acceptance criteria):
python scripts/benchmark.py --config configs/ohrc_nac.yaml \
  --use-msm --selector models/msm_v1.pkl

# Resume after interruption:
python scripts/benchmark.py --config configs/ohrc_nac.yaml --resume

# Single pair debug (MSM disabled explicitly):
python scripts/benchmark.py --pair <pair_id> --matcher rift2 --no-msm --keep-intermediate -v

# MSM-aware leakage audit:
python -m src.evaluation.leakage_audit --manifest data/pairs/manifest.jsonl --check-msm
```

---

## 4. Orchestration rules (v2.0 addition)

*(Rules 1–6 unchanged from BEST_PIPELINE.md)*

7. **MSM is never a hard dependency.** If `models/msm_v1.pkl` is missing or S4.5 throws an
   exception, the pipeline falls back to benchmark mode automatically and logs the event.
8. **MSM production mode is an opt-in flag** (`--use-msm`). It must not activate automatically
   on a fresh checkout that has no trained model.
9. **Every MSM routing decision is recorded** in `selector.json`. The fallback log
   `msm_fallback.jsonl` captures every post-S4.5 gate failure that triggered an M0 escalation.
10. **MSM model versioning:** the model file includes a version tag (`msm_v1`) that is written
    into every `selector.json`. If the model is retrained, the version is incremented and old
    `selector.json` files are not overwritten (they carry a stale version tag, which is auditable).

---

## 5. Failure & retry matrix (v2.0 additions)

| Symptom | Likely cause | Action |
|---|---|---|
| spiceinit fails at S1 | missing/mismatched CK kernel | Fetch ch2_att_*_v1.bc; re-run S1 |
| isisimport crash at S1 | product renamed | Restore original ISRO filename (F10) |
| ODE returns no NAC at S2 | footprint outside NAC coverage | Fall back to WAC crop |
| mask fraction >30% at S3 | extreme shadow (polar/low sun) | Keep pair, flag; matchers proceed on unmasked area |
| candidates <150 at S4 | textureless mare or GSD mismatch | Next matcher; record failure |
| coverage <0.60 at S5 | matches clustered | Relax cap once; if still low, matcher failed |
| inlier_ratio <0.05 at S6 | matcher noise / degenerate terrain | Widen t_gsd ×1.5; else tile-wise; else next matcher |
| refinement <70% at S7 | multimodal NCC peaks | Keep unrefined coords; flag pair |
| GPU OOM (M2) | large patches | Reduce kp_limit; run on CPU |
| **[NEW] S4.5: model file missing** | **MSM not yet trained** | **Log warning; fall back to benchmark_mode for all pairs** |
| **[NEW] S4.5: feature extraction error** | **L1 meta.json missing MSM fields** | **Re-run S3 with updated preprocess.py; or skip MSM for this pair** |
| **[NEW] S4 gate fails on MSM-routed matcher** | **Routing error or unusual pair** | **Auto-escalate to M0; log in msm_fallback.jsonl** |
| **[NEW] MSM benchmark: accuracy <70%** | **Insufficient training data or feature leakage** | **Do not activate production mode; check leakage audit; expand training set** |

---

## 6. Pilot-run checklist (v2.0 — MSM items added at end)

1. ☐ conda asp env active; `pip install lightgbm scikit-learn` done
2. ☐ ISISDATA/ALESPICEROOT exported; non-CK kernels fetched
3. ☐ CK kernel for strip date 2020-08-27 window fetched
4. ☐ Downloaded PRADAN/CHMAP: 2 OHRC strips + TMC-2 ortho/DEM — filenames untouched
5. ☐ S1 ingest → 3 products in products.jsonl with footprints + solar angles
6. ☐ S2 build_pairs → ≥ 3 manifest entries; `msm_split` field present
7. ☐ S3 preprocess → masks 5–30%; **meta.json includes all 6 MSM feature fields**
8. ☐ S4.5 selector (benchmark mode) → selector.json with `routing_reason: benchmark_mode`; msm_features.json present
9. ☐ S4–S7 for M0 on each pilot pair → geometry.json + matches_refined.json
10. ☐ S8 → registered.tif opens in QGIS; checkerboard QC aligned
11. ☐ S9 → leaderboard.csv row for M0; leakage audit passes
12. ☐ failures.jsonl reviewed; msm_fallback.jsonl empty (no fallback events for M0 which is always the safe path)
13. ☐ Repeat S4–S9 for rift2, lightglue, crater
14. ☐ **[MSM phase] Run `train_msm.py` after ≥ 50 labeled pairs exist**
15. ☐ **[MSM phase] `leakage_audit --check-msm` exits 0**
16. ☐ **[MSM phase] `benchmark.py --mode msm_eval` → msm_benchmark.csv; check accuracy ≥ 70% and RMSE degradation ≤ +0.10 px**
17. ☐ **[MSM phase] Only then: enable `--use-msm` in production runs**

---

## 7. Definition of done (pipeline — v2.0)

- All stages S0–S9 (including S4.5) runnable end-to-end from the CLI on a fresh checkout.
- `selector.json` and `msm_features.json` produced for every pair that passes S3.
- MSM benchmark: `msm_benchmark.csv` shows accuracy ≥ 70% AND RMSE degradation ≤ +0.10 px on test split, before MSM is activated in production mode.
- `leakage_audit --check-msm` passes (zero shared MSM train/test geo-cells).
- `msm_fallback.jsonl` exists and every entry is reviewed.
- All existing definition-of-done criteria from BEST_PIPELINE.md §7 still hold.

---

*Companion docs: ARCHITECTURE.md (design), FEATURES.md (MSM inputs), CONFIGURATION.md (config keys),
INTERFACES.md (contracts), VALIDATION.md (acceptance), DECISIONS.md (rationale).*
