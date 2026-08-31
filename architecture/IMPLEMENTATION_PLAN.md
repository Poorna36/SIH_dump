# SIH26166 — IMPLEMENTATION PLAN (MSM Addition, v1.0)

Phased, executable plan for implementing the Matcher Selection Model (MSM) layer described in
`ARCHITECTURE.md §L1.5`. This is the in-repository version of the design document. It is
updated as work progresses.

**Pre-condition:** Phases P0–P5 of the original architecture must be complete (full benchmark
with all 4 matchers on ≥ 50 pairs, `results/leaderboard.csv` exists, `data/pairs/manifest.jsonl`
has `split` fields set). The MSM is implemented in Phase P5.5 — it does not block any earlier phase.

---

## Phase P5.5 — MSM Implementation

### P5.5.0 — Prerequisites check

- [ ] `results/leaderboard.csv` exists with ≥ 50 rows where all applicable matchers succeeded.
- [ ] `data/pairs/manifest.jsonl` has `geo_cell` and `split` fields for all pairs.
- [ ] `pip install lightgbm scikit-learn pydantic` in the conda asp env.
- [ ] All existing tests pass (no regressions from P0–P5 code).

---

### P5.5.1 — Data layer extension (1–2 hours)

**Files to modify:**

#### `src/ingest/records.py`
- Add `msm_split: str` field to `PairRecord` dataclass.
- Default: `msm_split = split` (set in `build_pairs.py`).

#### `scripts/build_pairs.py`
- Write `msm_split` field to manifest (same value as `split` unless `--msm-split-override` is passed).

#### `scripts/preprocess.py`
- After computing shadow mask and normalization, compute and write 6 new fields to `meta.json`:
  - `masked_fraction` (fraction of pixels flagged)
  - `src_texture_contrast` (mean local σ in 8×8 windows)
  - `ref_texture_contrast`
  - `src_mean_gradient` (mean Sobel magnitude)
  - `ref_mean_gradient`
  - `tile_count`
- See FEATURES.md §2.2 for the exact computation.

**Test after this step:**
- Run `preprocess.py` on one pilot pair.
- Verify `meta.json` contains all 6 new fields.
- Run existing test suite — zero regressions.
- Run `build_pairs.py` on a test manifest entry — verify `msm_split` field present in output.

---

### P5.5.2 — Feature extraction module (2–3 hours)

**Files to create:**

#### `src/selector/__init__.py`
```python
# empty
```

#### `src/selector/features.py`
- Implement `FeatureVector` dataclass (INTERFACES.md §2.1).
- Implement `extract_from_pair_record(pr) -> dict`.
- Implement `extract_features(pr, l1_meta) -> FeatureVector`.
- Implement `FEATURE_NAMES`, `CATEGORICAL_FEATURES` constants.
- Raise `KeyError` if required L1 meta fields are missing.

**Tests to write (`tests/selector/test_features.py`):**
- All unit tests from VALIDATION.md §1.1.
- `test_feature_order` (to_array shape and order).
- `test_validate_contract_violations` (gsd_ratio=1.5 → ValueError).

---

### P5.5.3 — Config and interface layer (1–2 hours)

**Files to create:**

#### `src/selector/config.py`
- Implement `MSMConfig` and sub-configs using Pydantic v2 (INTERFACES.md §2.4).
- Implement `load_msm_config(path: str) -> MSMConfig`.
- Validators: `tau_high > tau_low >= 0.25`, composite weights sum to 1.0.

#### `configs/msm.yaml`
- Write the full annotated schema (CONFIGURATION.md §1).
- `msm.enabled: false` (never true at this stage).

#### `configs/matchers.yaml` (modify)
- Add `msm_safe_fallback: "sift"` and `msm_matcher_order: [sift, rift2, lightglue, crater]`.

**Tests to write (`tests/selector/test_config.py`):**
- `test_valid_config_loads` (default msm.yaml parses without error).
- `test_tau_order_validator` (tau_high <= tau_low → ValidationError).
- `test_weights_sum_validator` (weights ≠ 1.0 → ValidationError).

---

### P5.5.4 — MatcherSelector model wrapper (3–4 hours)

**Files to create:**

#### `src/selector/model.py`
- Implement `MatcherSelector` class (INTERFACES.md §2.3):
  - `__init__(model_path, stats_path, cfg)`: load model; raise `FileNotFoundError` if enabled and missing.
  - `route(pair_record, l1_meta) -> SelectorResult`: full routing logic.
  - `_apply_hard_rules(probs, pair_record) -> np.ndarray`.
  - `_check_distribution_shift(fv) -> list[str]`.
- Implement `SelectorResult` dataclass (INTERFACES.md §2.2).
- Implement `MATCHER_NAMES`, `ROUTING_REASONS` constants.
- All exceptions in `route()` must be caught; never propagate to caller.

**Tests to write (`tests/selector/test_model.py`):**
- All unit tests from VALIDATION.md §1.2 (hard-rule masking).
- All unit tests from VALIDATION.md §1.3 (routing logic).
- All unit tests from VALIDATION.md §1.4 (SelectorResult invariants).
- All unit tests from VALIDATION.md §1.5 (model load / unavailability).
- All unit tests from VALIDATION.md §1.6 (JSON round-trip).

Note: unit tests for routing logic can use a mock model (always returns uniform probabilities)
so they run without a trained model file.

---

### P5.5.5 — Pipeline integration (2–3 hours)

**Files to modify:**

#### `scripts/benchmark.py`
- Add `--use-msm` flag and `--no-msm` flag (default: `--no-msm` until explicitly enabled).
- Add `--mode msm_eval` flag for MSM benchmarking.
- Between S3 and S4: instantiate `MatcherSelector` once; call `selector.route()` per pair;
  write `selector.json` and `msm_features.json`; pass `matchers_to_run` to registry loop.
- On S4 gate failure: if routed matcher fails, escalate to `cfg.fallback.on_s4_gate_failure`
  (default "sift"); write event to `results/msm_fallback.jsonl`.
- Orchestration rule: MSM is never a hard dependency — any exception falls back to benchmark_mode.

#### `src/evaluation/leakage_audit.py`
- Add `--check-msm` flag.
- Implement checks from VALIDATION.md §3:
  1. MSM train/test geo-cell disjointness.
  2. Label source disjointness.
  3. Feature disjointness consistency.
  4. Cross-check with benchmark split.

**Tests to write (`tests/test_pipeline_integration.py`):**
- Smoke test: run `benchmark.py` on 1 pilot pair with `--no-msm` (benchmark_mode) — selector.json
  written with `routing_reason: benchmark_mode`; existing pipeline output unchanged.
- Smoke test: run `benchmark.py` on 1 pilot pair with `--use-msm` (mock model) — only 1 matcher
  runs when confidence is high; fallback fires when S4 gate fails.

---

### P5.5.6 — Training script (3–4 hours)

**Files to create:**

#### `src/selector/train.py`
Core training logic (called by `scripts/train_msm.py`):
- `load_labels(leaderboard_csv, manifest, split="train") -> pd.DataFrame`
  - Joins leaderboard with manifest on `pair_id`.
  - Filters to `msm_split == "train"`.
  - Filters to pairs where all applicable matchers succeeded.
  - Computes composite score per (pair × matcher):
    `score = 0.5*(1/rmse_norm) + 0.25*inlier_ratio + 0.25*spatial_coverage`
    where `rmse_norm = rmse_px / max(rmse_px for that pair)`.
  - Labels each pair with the argmax matcher index (0–3 per `msm_matcher_order`).
- `load_feature_vectors(manifest, processed_dir, pair_ids) -> pd.DataFrame`
  - Reads `meta.json` for each pair; assembles FeatureVector.
- `train(features_df, labels_df, cfg) -> (model, stats_dict)`
  - LightGBM multi-class classifier with cross-validation.
  - Returns fitted sklearn Pipeline and training distribution stats.
- `evaluate_cv(model, X, y) -> dict`
  - Returns CV accuracy, per-class accuracy, log-loss.

#### `scripts/train_msm.py`
CLI wrapper:
```bash
python scripts/train_msm.py \
  --leaderboard results/leaderboard.csv \
  --manifest data/pairs/manifest.jsonl \
  --processed data/processed \
  --split msm_split --split-value train \
  --out models/msm_v1.pkl \
  --config configs/msm.yaml \
  --plot-importance results/figures/msm_feature_importance.png
```
- Checks `min_labeled_pairs` before training (exits with informative error if not met).
- Writes `models/msm_v1.pkl` and `models/msm_v1_stats.json`.
- Writes `results/figures/msm_feature_importance.png`.
- Prints CV accuracy and class distribution to stdout.

**Tests to write (`tests/selector/test_train.py`):**
- `test_label_generation_winner_is_argmax` — synthetic leaderboard; verify label = highest-score matcher.
- `test_label_generation_filters_split` — test-split pairs must not appear in training labels.
- `test_min_pairs_guard` — if N < min_labeled_pairs, raise informative error.
- `test_train_smoke` — train on 10 synthetic rows; model file written; no exception.

---

### P5.5.7 — MSM evaluation module (2–3 hours)

**Files to create:**

#### `src/evaluation/msm_eval.py`
- `compute_msm_metrics(benchmark_csv, leaderboard_csv) -> dict`
  - Computes all 7 quantitative metrics from VALIDATION.md §2.3.
  - Returns dict with metric names as keys.
- `print_summary(metrics, thresholds, out_path)` — writes `msm_benchmark_summary.txt`.
- `per_stratum_breakdown(benchmark_csv, manifest) -> pd.DataFrame`.

Integrated into `scripts/benchmark.py --mode msm_eval` (see P5.5.5).

**Tests (`tests/evaluation/test_msm_eval.py`):**
- `test_perfect_selector` — when MSM always picks the oracle: accuracy=100%, degradation=0.
- `test_worst_selector` — when MSM always picks the worst: verify degradation is positive.
- `test_thresholds_applied_correctly` — mock metrics dict; PASS/FAIL assigned correctly.

---

### P5.5.8 — Final validation and activation (0.5 hours)

1. Run full VALIDATION.md §5 checklist.
2. If all 8 acceptance criteria pass: set `msm.enabled: true` in `configs/msm.yaml`.
3. Record activation in DECISIONS.md (see entry DEC-007).
4. Tag the git commit as `msm-v1-activated`.

---

## Timeline estimate

| Sub-phase | Estimated effort |
|---|---|
| P5.5.0 Prerequisites | 0.5 h |
| P5.5.1 Data layer extension | 1–2 h |
| P5.5.2 Feature extraction module | 2–3 h |
| P5.5.3 Config and interface layer | 1–2 h |
| P5.5.4 MatcherSelector model wrapper | 3–4 h |
| P5.5.5 Pipeline integration | 2–3 h |
| P5.5.6 Training script | 3–4 h |
| P5.5.7 MSM evaluation module | 2–3 h |
| P5.5.8 Final validation + activation | 0.5 h |
| **Total** | **~15–22 hours** |

---

## Dependencies and sequencing

```
P5.5.0 → P5.5.1 → P5.5.2 → P5.5.3 → P5.5.4 → P5.5.5
                                              ↓
                                          P5.5.6 (requires labeled data from P0–P5)
                                              ↓
                                          P5.5.7
                                              ↓
                                          P5.5.8
```

P5.5.2, P5.5.3, and P5.5.4 can be developed in parallel after P5.5.1.
P5.5.6 requires both trained data (from P0–P5) and the model wrapper (P5.5.4) to be complete.

---

## Definition of done for P5.5

- [ ] All unit tests in VALIDATION.md §1 pass with zero failures.
- [ ] `preprocess.py` writes all 6 MSM feature fields to `meta.json` on every pair.
- [ ] `train_msm.py` completes without error on the available train pairs.
- [ ] `leakage_audit --check-msm` exits 0.
- [ ] `benchmark.py --mode msm_eval` produces `msm_benchmark.csv` and `msm_benchmark_summary.txt`.
- [ ] All 8 acceptance criteria (VALIDATION.md §2.3) show PASS.
- [ ] `configs/msm.yaml` updated to `enabled: true`.
- [ ] DECISIONS.md updated with DEC-007 (MSM activation).
- [ ] Git tag `msm-v1-activated` pushed.
