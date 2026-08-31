# SIH26166 — VALIDATION (MSM Acceptance Protocol, v1.0)

Defines all acceptance tests, benchmarking protocol, leakage audit extension, and minimum
performance thresholds that must be met before the Matcher Selection Model (MSM) is activated
in production mode (`msm.enabled: true`).

**Critical rule:** The MSM is never activated before passing every criterion in §2.
Until activation, it runs in `benchmark_mode` (transparent pass-through) and the pipeline
behaves identically to v1.0.

Related docs: ARCHITECTURE.md §L1.5 & §16-addition, PIPELINE.md §S4.5 & §S9, FEATURES.md §5,
CONFIGURATION.md §1, INTERFACES.md §2.

---

## 1. Unit tests (run on every code change via CI)

### 1.1 FeatureVector extraction

| Test | Input | Expected |
|---|---|---|
| `test_extract_features_ohrc_nac` | Synthetic PairRecord (OHRC→NAC) + valid meta.json | Returns FeatureVector; `sensor_pair_enc==0`; all float fields finite; `validate()` passes |
| `test_extract_features_tmc_wac` | Synthetic PairRecord (TMC→WAC) + valid meta.json | `sensor_pair_enc==1`; `validate()` passes |
| `test_extract_features_missing_l1_field` | meta.json with `masked_fraction` removed | Raises `KeyError`; caller fallback path exercised |
| `test_crater_density_log1p` | `crater_density=0.0` | `fv.crater_density == 0.0` (log1p(0)=0) |
| `test_gsd_ratio_contract` | `gsd_ratio=1.5` (violates contract) | `validate()` raises `ValueError` |
| `test_feature_order` | Any valid FeatureVector | `fv.to_array().shape == (13,)` and values match `FEATURE_NAMES` order exactly |

### 1.2 Hard-rule masking

| Test | Setup | Expected |
|---|---|---|
| `test_crater_gate_low_density` | `crater_density=2.0 < tau_c=5.0`; M3 initially has max prob | After masking, M3 prob == 0; remaining probs sum to 1.0 ± 1e-6; selected_matcher ≠ "crater" |
| `test_crater_gate_high_density` | `crater_density=10.0 >= tau_c=5.0` | M3 prob unchanged; masking does not zero it |
| `test_gpu_gate_no_gpu` | `gpu_available=False`; M2 initially has max prob | After masking, M2 prob == 0; selected_matcher ≠ "lightglue" |
| `test_gpu_gate_with_gpu` | `gpu_available=True` | M2 prob unchanged |
| `test_iirs_track_gate` | `sensor_pair_enc=2` (IIRS→WAC) | M2 and M3 probs == 0; selected ∈ {"sift", "rift2"} |
| `test_all_gates_applied_simultaneously` | No GPU + low crater density | Both M2 and M3 zeroed; probs sum to 1.0; selected ∈ {"sift", "rift2"} |
| `test_degenerate_all_zeroed` | Hypothetical: all matchers gated | Returns uniform over non-gated; M0 (sift) always available (degenerate safeguard) |

### 1.3 Routing logic

| Test | `confidence` | Expected `routing_reason` | `matchers_to_run` |
|---|---|---|---|
| `test_route_high_confidence` | 0.80 >= tau_high=0.65 | `"high_confidence"` | `[selected]` (length 1) |
| `test_route_low_confidence_fallback` | 0.52 ∈ [tau_low, tau_high) | `"low_confidence_fallback"` | `[fallback]` (length 1) |
| `test_route_very_low_confidence` | 0.30 < tau_low=0.40 | `"very_low_confidence_full_mode"` | all applicable matchers |
| `test_route_benchmark_mode` | any (msm.enabled=false) | `"benchmark_mode"` | all applicable matchers |

### 1.4 SelectorResult invariants

```python
def test_selector_result_invariants(result: SelectorResult):
    assert abs(sum(result.all_probs.values()) - 1.0) < 1e-6
    assert result.selected_matcher == max(result.all_probs, key=result.all_probs.get)
    assert result.confidence == result.all_probs[result.selected_matcher]
    assert result.routing_reason in ROUTING_REASONS
    assert len(result.matchers_to_run) >= 1
    if result.routing_reason == "high_confidence":
        assert result.matchers_to_run == [result.selected_matcher]
```

### 1.5 Model load / unavailability

| Test | Condition | Expected |
|---|---|---|
| `test_model_not_found_enabled_false` | `msm.enabled=False`; model file missing | `__init__` succeeds; `route()` returns `benchmark_mode` result |
| `test_model_not_found_enabled_true` | `msm.enabled=True`; model file missing | `__init__` raises `FileNotFoundError` at startup (fail fast) |
| `test_route_no_exception_on_corrupt_model` | Model file is corrupt (pickle error) | `route()` catches exception; returns `model_unavailable` result with all matchers |

### 1.6 JSON serialization round-trip

```python
def test_selector_result_roundtrip(result: SelectorResult):
    d   = result.to_json()
    rt  = SelectorResult.from_json(d)
    assert rt == result
```

### 1.7 L1 meta.json backward compatibility

| Test | Condition | Expected |
|---|---|---|
| `test_old_meta_json_no_msm_fields` | meta.json without the 6 new fields | Feature extraction raises `KeyError`; pipeline logs warning and falls back to benchmark_mode |
| `test_preprocess_writes_all_msm_fields` | Run `preprocess.py` on any pilot pair | All 6 MSM field names present in output meta.json; all values finite |

---

## 2. MSM benchmark protocol (must pass before production activation)

### 2.1 Prerequisites

Before running the MSM benchmark:
1. Full multi-matcher benchmark (all 4 matchers × all test pairs) must have completed successfully.
2. `results/leaderboard.csv` must contain at least 50 labeled **train-split** pairs where all
   applicable matchers produced valid RegistrationResult entries.
3. `train_msm.py` has been run and `models/msm_v1.pkl` + `models/msm_v1_stats.json` exist.
4. **Leakage audit must pass** (§3) before any benchmark number is quoted.

### 2.2 Benchmark command

```bash
# Step 1: Train the MSM
python scripts/train_msm.py \
  --leaderboard results/leaderboard.csv \
  --manifest data/pairs/manifest.jsonl \
  --processed data/processed \
  --split msm_split --split-value train \
  --out models/msm_v1.pkl \
  --config configs/msm.yaml \
  --plot-importance results/figures/msm_feature_importance.png

# Step 2: Extended leakage audit
python -m src.evaluation.leakage_audit \
  --manifest data/pairs/manifest.jsonl --check-msm
# Must exit 0 before proceeding.

# Step 3: MSM benchmark on test split
python scripts/benchmark.py \
  --config configs/ohrc_nac.yaml \
  --splits test \
  --mode msm_eval \
  --selector models/msm_v1.pkl \
  --out results/msm_benchmark.csv

# Step 4: Print aggregate metrics
python -m src.evaluation.msm_eval \
  --benchmark results/msm_benchmark.csv \
  --print-summary
```

### 2.3 Acceptance criteria (ALL must pass)

| # | Metric | Threshold | Rationale |
|---|---|---|---|
| AC1 | **Selector accuracy** (% test pairs where MSM picked the true best matcher) | **≥ 70%** | Below 70%, the selector is no better than random among 4 classes (25%) by a very wide margin, but still provides value; 70% is the minimum justified by the RMSE degradation bound |
| AC2 | **Top-2 accuracy** (% test pairs where true best is in {selected, fallback}) | **≥ 85%** | Ensures the fallback_matcher is nearly always correct even when first choice is wrong |
| AC3 | **Mean RMSE degradation** (mean over test pairs of: MSM-selected RMSE − oracle-best RMSE) | **≤ +0.10 px** | Acceptable accuracy penalty for 3–4× runtime reduction; larger than the L5 refinement gain would be unacceptable |
| AC4 | **Max RMSE degradation on any single test pair** | **≤ +0.50 px** | Prevents catastrophic misrouting on any individual pair |
| AC5 | **Runtime reduction** (mean % reduction vs all-matchers baseline) | **≥ 50%** | The MSM's primary purpose; if <50%, the selector is saving less than running 2 of the 4 matchers, which the rule-based policy already achieves |
| AC6 | **Fallback rate** (% of test pairs with `routing_reason == very_low_confidence_full_mode`) | **≤ 20%** | Too many full-mode pairs defeats the purpose |
| AC7 | **Feature importance non-trivial** | All top-5 features have >0 LightGBM gain | Guards against a degenerate model that ignores most features |
| AC8 | **Leakage audit** (`--check-msm`) | **Exit 0** | Mandatory pre-condition (F15) |

### 2.4 Reporting format

The MSM benchmark report is written to `results/msm_benchmark_summary.txt`:

```
MSM Benchmark Summary — msm_v1
================================
Test split pairs evaluated:  N
Pairs with oracle label:      N   (all 4 matchers succeeded)

Selector accuracy:            XX.X %    [threshold: ≥ 70.0%]   PASS / FAIL
Top-2 accuracy:               XX.X %    [threshold: ≥ 85.0%]   PASS / FAIL
Mean RMSE degradation:       +X.XX px   [threshold: ≤ +0.10 px] PASS / FAIL
Max RMSE degradation:        +X.XX px   [threshold: ≤ +0.50 px] PASS / FAIL
Runtime reduction (mean):     XX.X %    [threshold: ≥ 50.0%]   PASS / FAIL
Fallback rate:                XX.X %    [threshold: ≤ 20.0%]   PASS / FAIL
Feature importance non-trivial:          [see figure]            PASS / FAIL
Leakage audit (--check-msm):             exit 0                  PASS / FAIL

OVERALL: PASS (activate msm.enabled: true) / FAIL (do not activate)

Per-stratum breakdown:
  Latitude 0–30°:   accuracy XX.X%, RMSE degradation +X.XX px
  Latitude 30–55°:  accuracy XX.X%, RMSE degradation +X.XX px
  Latitude >55°:    accuracy XX.X%, RMSE degradation +X.XX px
  Δ-azimuth <30°:   accuracy XX.X%, RMSE degradation +X.XX px
  Δ-azimuth 30–90°: accuracy XX.X%, RMSE degradation +X.XX px
  Δ-azimuth >90°:   accuracy XX.X%, RMSE degradation +X.XX px
  Highland:         accuracy XX.X%, RMSE degradation +X.XX px
  Maria:            accuracy XX.X%, RMSE degradation +X.XX px
  Polar:            accuracy XX.X%, RMSE degradation +X.XX px
```

Per-stratum rows are mandatory — the same rule that forbids hiding polar rows in the main
leaderboard (F13) applies here. A stratum with ≤ 3 test pairs is reported but not counted
toward the overall PASS/FAIL.

### 2.5 What to do if any criterion fails

| Criterion fails | Root cause investigation | Remediation |
|---|---|---|
| AC1/AC2 (accuracy) | Check confusion matrix; which matcher is most mis-predicted? | Expand training set; check for class imbalance (oversample minority class); review feature importance for leakage |
| AC3/AC4 (RMSE degradation) | Which pairs have worst degradation? Are they in a specific stratum? | Add stratum-specific features; or add a hard rule for that stratum |
| AC5 (runtime reduction) | High fallback rate or many low-confidence pairs | Lower `tau_low` to reduce full-mode pairs; check if confidence is well-calibrated (Brier score) |
| AC6 (fallback rate) | Model is poorly calibrated; too many uncertain predictions | Check training data: are all 4 matchers nearly tied on most pairs? (implies the task is harder than expected) |
| AC7 (feature importance) | Degenerate model | Check for constant-value features in training data |
| AC8 (leakage) | Shared geo-cells between MSM train/test | Re-assign `msm_split` field; never let the geo-cell that produced a label also appear in MSM test |

**Do not lower the acceptance thresholds to make a failing model pass.** If the model cannot
meet the thresholds with the available data, keep `msm.enabled: false` and use the rule-based
arbitration policy (ARCHITECTURE.md §5).

---

## 3. Leakage audit extension

The existing `leakage_audit` script is extended with a `--check-msm` flag:

```bash
python -m src.evaluation.leakage_audit \
  --manifest data/pairs/manifest.jsonl --check-msm
```

**Checks performed:**

1. **MSM split disjointness**: zero geo-cells appear in both MSM train and MSM test sets.
   `{geo_cell : msm_split == "train"} ∩ {geo_cell : msm_split == "test"} == ∅`

2. **Label source disjointness**: every pair used as an MSM training label (i.e., used to define
   the winning matcher from the leaderboard) has `msm_split == "train"`. No test-split pair's
   RegistrationResult was used to define a training label.

3. **Feature disjointness consistency**: no pair with `msm_split == "test"` appears in the
   LightGBM training data (checked against the training log written by `train_msm.py`).

4. **Cross-check with benchmark split**: for every pair, `msm_split` must equal `split` unless
   an explicit override was recorded in the manifest. Mixed-split pairs (split=train, msm_split=test)
   are permitted for the secondary hold-out but must be intentional.

**Exit codes:**
- `0` — all checks pass.
- `1` — leakage detected (prints which pairs and which geo-cells are affected).
- `2` — manifest unreadable or missing required fields.

---

## 4. Ongoing validation (post-activation)

After MSM is activated in production (`msm.enabled: true`), the following checks are run
periodically (e.g., after every batch of 10 new pairs processed):

| Check | Frequency | Action on failure |
|---|---|---|
| `msm_fallback.jsonl` size | Per batch | If fallback rate > 25% in the new batch, log alert; consider reverting to `benchmark_mode` |
| Distribution shift warning count | Per batch | If >30% of new pairs trigger out-of-range warnings, the new pairs may be from an unseen stratum; retrain MSM |
| RMSE delta (new pairs, where GT available) | Monthly | If mean RMSE degradation on new GT pairs > +0.15 px, trigger retraining |
| Model version consistency | On startup | If `selector.json` files from recent runs have a different `selector_version` than the loaded model, alert |

---

## 5. Summary checklist (before activating msm.enabled: true)

```
☐ All 8 unit test groups (§1) pass with zero failures
☐ train_msm.py completed successfully; models/msm_v1.pkl exists
☐ leakage_audit --check-msm exits 0
☐ msm_benchmark.csv generated on test split
☐ All 8 acceptance criteria (AC1–AC8) show PASS in msm_benchmark_summary.txt
☐ Per-stratum breakdown reviewed — no stratum shows RMSE degradation > +0.30 px
☐ Feature importance plot reviewed — no degenerate model
☐ msm.yaml updated: enabled: true, model_version: msm_v1
☐ The activation is recorded in DECISIONS.md with the benchmark summary date
```
