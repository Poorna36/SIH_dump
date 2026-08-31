# SIH26166 — CONFIGURATION (v1.0)

Configuration reference for the Matcher Selection Model (MSM / L1.5) and its integration with
the existing pipeline configuration. All pre-existing config keys (`ohrc_nac.yaml`, `tmc_wac.yaml`,
`matchers.yaml`) are unchanged; this document defines the new `configs/msm.yaml` file and the
minimal additions to `configs/matchers.yaml`.

Related docs: ARCHITECTURE.md §L1.5, PIPELINE.md §S4.5, FEATURES.md, INTERFACES.md, VALIDATION.md.

---

## 1. `configs/msm.yaml` (NEW)

Full annotated schema:

```yaml
# configs/msm.yaml
# Matcher Selection Model (MSM) configuration.
# All threshold values are configurable here; the defaults are the validated baseline values.

msm:
  # ─── Enable / disable ──────────────────────────────────────────────────────
  enabled: false
  # Set to true ONLY after msm_benchmark.csv shows:
  #   - selector_accuracy >= 0.70
  #   - mean_rmse_degradation_px <= 0.10
  # on the test split. Until then, keep false (benchmark/training mode).

  # ─── Model artifact ────────────────────────────────────────────────────────
  model_path: "models/msm_v1.pkl"
  # Path to the trained LightGBM model file (sklearn Pipeline wrapper).
  # If the file does not exist at runtime, MSM falls back to benchmark mode
  # for all pairs (logged as a warning, never an error).

  model_stats_path: "models/msm_v1_stats.json"
  # Training distribution statistics used for out-of-range warnings.
  # Format: {feature_name: {min, max, mean, std}} for each continuous feature.

  model_version: "msm_v1"
  # Written into every selector.json. Increment when the model is retrained.

  # ─── Confidence thresholds ──────────────────────────────────────────────────
  tau_high: 0.65
  # If max(all_probs) >= tau_high:
  #   run only the selected_matcher (high-confidence routing).
  # Range: (tau_low, 1.0). Do not set above 0.80 without re-validating on test data.

  tau_low: 0.40
  # If tau_low <= max(all_probs) < tau_high:
  #   run the fallback_matcher (2nd-highest probability after hard-rule masking).
  # Range: (0.25, tau_high). Below this both matchers may be uncertain.
  # If max(all_probs) < tau_low: revert to full multi-matcher mode for this pair.

  # ─── Hard-rule overrides ────────────────────────────────────────────────────
  # These rules are applied by zeroing the relevant probability and renormalizing.
  # They mirror the existing arbitration policy (ARCHITECTURE.md §5) and are
  # always active, regardless of the confidence thresholds.

  hard_rules:
    crater_density_gate:
      enabled: true
      tau_c: 5.0                  # craters/Mpx; same as matchers.yaml crater.params.tau_c
      # M3 (crater) probability is set to 0 if crater_density < tau_c in either image.

    gpu_gate:
      enabled: true
      check_at_startup: true
      # M2 (lightglue) probability is set to 0 if no GPU is detected.
      # gpu_available is determined once at pipeline startup (torch.cuda.is_available()).

    iirs_track_gate:
      enabled: true
      # If sensor_pair == IIRS_WAC, M2 and M3 probabilities are set to 0.
      # IIRS uses a separate pipeline track (F12); learned/crater matchers are not validated
      # for the IIRS spectral domain.

  # ─── Fallback policy ────────────────────────────────────────────────────────
  fallback:
    on_model_load_error: "benchmark_mode"
    # Options: "benchmark_mode" (run all matchers) | "rule_based" (use §5 arbitration policy)
    # "benchmark_mode" is the safer default.

    on_feature_extraction_error: "benchmark_mode"
    # If L1 meta.json is missing MSM fields, fall back to benchmark_mode for that pair.

    on_s4_gate_failure: "sift"
    # If the MSM-routed matcher fails S4 (< 150 candidates), automatically try this matcher.
    # "sift" = M0, which is always the safe last-resort fallback.
    # This event is logged in results/msm_fallback.jsonl.

    log_all_fallback_events: true
    # Whether to write every fallback event to results/msm_fallback.jsonl.

  # ─── Inference logging ──────────────────────────────────────────────────────
  logging:
    write_selector_json: true
    # Always write results/<pair_id>/selector.json (recommended; disable only to save disk space).

    write_feature_vector_json: true
    # Write results/<pair_id>/msm_features.json (raw FeatureVector for audit / retraining).

    warn_on_out_of_range_features: true
    # Log a warning if any continuous feature falls outside training distribution ± 2σ.

  # ─── Training configuration ─────────────────────────────────────────────────
  # Used by scripts/train_msm.py (not used at inference time).
  training:
    min_labeled_pairs: 50
    # Minimum number of labeled pairs (all matchers succeeded) before training is attempted.
    # If fewer pairs are available, train_msm.py exits with an informative error.

    composite_score_weights:
      rmse_norm_inv: 0.50       # weight for (1 / normalized_RMSE)
      inlier_ratio:  0.25       # weight for inlier_ratio
      spatial_coverage: 0.25   # weight for spatial_coverage
    # These weights define the "winner" label for each pair from the leaderboard.
    # Sum must equal 1.0.

    lightgbm_params:
      objective: "multiclass"
      num_class: 4
      metric: "multi_logloss"
      n_estimators: 200
      learning_rate: 0.05
      max_depth: 6
      num_leaves: 31
      min_child_samples: 5
      subsample: 0.8
      colsample_bytree: 0.8
      random_state: 42
      n_jobs: -1
    # Default hyperparameters. Cross-validated on the train split.
    # Final model is retrained on all train pairs with these params.

    cross_validation:
      n_splits: 5
      strategy: "StratifiedKFold"
      # Stratified by the winning matcher class to ensure all 4 classes are represented
      # in each fold. Geographic stratification (geo-cell level) is enforced by the
      # msm_split field in the manifest — not by the CV strategy itself.

    feature_importance_plot: "results/figures/msm_feature_importance.png"
    # Path to write the feature importance bar chart after training.
```

---

## 2. Additions to `configs/matchers.yaml`

The MSM references matcher names by their registry key. The following field is added to the
`arbitration` section to declare which matcher is the universal safe fallback when all else fails:

```yaml
# configs/matchers.yaml (addition only — all existing fields unchanged)
registry:
  sift:      {class: matching.sift.SiftMatcher}
  rift2:     {class: matching.rift.RiftMatcher,      params: {Ns: 4, No: 6, octaves: 3}}
  lightglue: {class: matching.lightglue.LightGlueMatcher, params: {nms: 3, kp_limit: 2048}}
  crater:    {class: matching.crater.CraterMatcher,  params: {K: 6, tau_c: 5}}

arbitration:
  order: [crater, lightglue, rift2, sift]
  fallback_floor: {inlier_ratio: 0.05, min_inliers: 20}
  msm_safe_fallback: "sift"            # ← NEW: the matcher MSM escalates to on any gate failure
  msm_matcher_order: [sift, rift2, lightglue, crater]  # ← NEW: integer index order used by MSM
```

`msm_matcher_order` defines the canonical mapping between LightGBM class index and matcher name:
- Index 0 → `sift`
- Index 1 → `rift2`
- Index 2 → `lightglue`
- Index 3 → `crater`

This mapping is fixed and must not change after any training run. If the registry gains a new
matcher, a new MSM version (msm_v2) must be trained.

---

## 3. Per-sensor-pair config (no changes to existing files)

`configs/ohrc_nac.yaml`, `configs/tmc_wac.yaml`, and `configs/iirs_wac.yaml` are **unchanged**.
The MSM reads sensor pair identity from the `PairRecord.source.sensor` and
`PairRecord.reference.sensor` fields, not from the config file name. No per-sensor-pair MSM
override is needed because sensor pair is already a feature (`sensor_pair_enc`) learned by the model.

---

## 4. Environment variables

| Variable | Purpose | Default |
|---|---|---|
| `MSM_MODEL_PATH` | Override `model_path` at runtime without editing configs | — (uses config) |
| `MSM_ENABLED` | `"1"` / `"0"` — runtime on/off override | — (uses config) |
| `MSM_LOG_LEVEL` | `"DEBUG"` / `"INFO"` / `"WARNING"` | `"INFO"` |

These are useful for CI runs and per-job overrides without editing config files.

---

## 5. Config validation

`scripts/benchmark.py` validates `configs/msm.yaml` on startup using a Pydantic schema
(defined in `src/selector/config.py`). Validation checks:

- `tau_low < tau_high < 1.0`
- `tau_low >= 0.25`
- `composite_score_weights` sum to 1.0 ± 1e-6
- `model_path` exists if `msm.enabled = true` (error, not warning — must not silently degrade)
- `msm_matcher_order` contains exactly the 4 registry keys, no duplicates
- `hard_rules.crater_density_gate.tau_c` matches `matchers.yaml crater.params.tau_c`
  (mismatch = warning; they can legitimately differ but must be intentional)

---

## 6. Config change log

| Version | Date | Change |
|---|---|---|
| 1.0 | 2026-08-31 | Initial MSM config schema |

All config changes that affect MSM training (composite score weights, LightGBM params, feature
set) require the model to be retrained and the version to be incremented. Config changes that
only affect inference thresholds (`tau_high`, `tau_low`) do not require retraining but must be
re-validated on the test split.
