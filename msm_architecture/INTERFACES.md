# SIH26166 — INTERFACES (v1.0)

Defines the data contracts and interfaces for all MSM (L1.5) components. The existing
`Matcher` interface (ARCHITECTURE.md §3) is reproduced here for completeness but is unchanged.
Every new contract in this document integrates with the existing pipeline without modifying any
existing interface.

Related docs: ARCHITECTURE.md §3–4, FEATURES.md, CONFIGURATION.md, PIPELINE.md §S4.5.

---

## 1. Existing interfaces (unchanged)

### 1.1 Matcher (L2 pluggable interface)

```python
# src/matching/base.py  — UNCHANGED
from abc import ABC, abstractmethod

class Matcher(ABC):
    name: str   # registry key: "sift" | "rift2" | "lightglue" | "crater"

    @abstractmethod
    def detect_and_describe(self, img: np.ndarray,
                             valid_mask: np.ndarray | None = None) -> "Keypoints":
        """
        Returns:
            Keypoints: dataclass with fields
                xy          : np.ndarray  shape (N, 2), float32, pixel coords
                scales      : np.ndarray  shape (N,),   float32
                orientations: np.ndarray  shape (N,),   float32, radians
                descriptors : np.ndarray  shape (N, D), float32
        Respects valid_mask (F5): keypoints within masked pixels are suppressed.
        """
        ...

    @abstractmethod
    def match(self, kp_src: "Keypoints", kp_ref: "Keypoints",
              cfg: dict) -> "MatchSet":
        """
        Returns:
            MatchSet: dataclass with fields
                src_indices : np.ndarray  shape (M,), int32
                ref_indices : np.ndarray  shape (M,), int32
                confidences : np.ndarray  shape (M,), float32, values in [0, 1]
        """
        ...

    def supports_semidense(self) -> bool:
        return False
```

### 1.2 PairRecord (data/pairs/manifest.jsonl)

```python
# src/ingest/records.py  — UNCHANGED (v2.0 adds msm_split field only)
@dataclass
class IllumRecord:
    src_incidence:  float
    src_azimuth:    float
    ref_incidence:  float
    ref_azimuth:    float
    delta_azimuth:  float

@dataclass
class SourceRecord:
    sensor:     str   # "OHRC" | "TMC" | "IIRS"
    product_id: str
    path:       str

@dataclass
class PairRecord:
    pair_id:          str
    source:           SourceRecord
    reference:        SourceRecord
    bbox_pad_deg:     list[float]   # [lon_min, lat_min, lon_max, lat_max]
    illum:            IllumRecord
    terrain_class:    str           # "highland" | "maria" | "polar" | "mixed"
    crater_density:   float         # craters / Mpx
    overlap_fraction: float
    gsd_ratio:        float         # src_gsd / ref_gsd
    geo_cell:         str           # e.g. "20E_15S"
    split:            str           # "train" | "test"
    msm_split:        str           # "train" | "test"  ← NEW in v2.0 (defaults to split)
```

### 1.3 RegistrationResult (results/pair_results/<pair_id>/<matcher>.json)

```python
# src/evaluation/records.py  — UNCHANGED
@dataclass
class RegistrationResult:
    pair_id:          str
    matcher:          str
    n_candidates:     int
    n_inliers:        int
    inlier_ratio:     float
    model:            str      # "similarity"|"affine"|"homography"|"tilewise_affine"
    rmse_px:          float
    pct_lt_1px:       float
    pct_lt_0p5px:     float
    medae_px:         float
    spatial_coverage: float
    grid_density_std: float
    runtime_s:        float
    refined:          bool
```

---

## 2. New interfaces (v2.0 — MSM)

### 2.1 FeatureVector

```python
# src/selector/features.py

FEATURE_NAMES = [
    "sensor_pair_enc",      # int  — see FEATURES.md §1
    "gsd_ratio",            # float
    "latitude_abs",         # float
    "delta_solar_azimuth",  # float
    "terrain_class_enc",    # int
    "crater_density",       # float (log1p-transformed)
    "masked_fraction",      # float
    "overlap_fraction",     # float
    "src_texture_contrast", # float
    "ref_texture_contrast", # float
    "src_mean_gradient",    # float
    "ref_mean_gradient",    # float
    "tile_count",           # int
]

CATEGORICAL_FEATURES = ["sensor_pair_enc", "terrain_class_enc"]

@dataclass
class FeatureVector:
    sensor_pair_enc:      int
    gsd_ratio:            float
    latitude_abs:         float
    delta_solar_azimuth:  float
    terrain_class_enc:    int
    crater_density:       float
    masked_fraction:      float
    overlap_fraction:     float
    src_texture_contrast: float
    ref_texture_contrast: float
    src_mean_gradient:    float
    ref_mean_gradient:    float
    tile_count:           int

    def to_array(self) -> np.ndarray:
        """Returns a float32 array in FEATURE_NAMES order for model input."""
        return np.array([getattr(self, f) for f in FEATURE_NAMES], dtype=np.float32)

    def to_dict(self) -> dict:
        return {f: getattr(self, f) for f in FEATURE_NAMES}

    def validate(self) -> None:
        """Raises ValueError on obviously invalid values."""
        assert 0 <= self.sensor_pair_enc <= 2,   "sensor_pair_enc out of range"
        assert 0 < self.gsd_ratio <= 1.0,        "gsd_ratio must be in (0, 1]"
        assert 0 <= self.latitude_abs <= 90,      "latitude_abs out of range"
        assert 0 <= self.delta_solar_azimuth <= 180, "delta_solar_azimuth out of range"
        assert 0 <= self.terrain_class_enc <= 3, "terrain_class_enc out of range"
        assert self.crater_density >= 0,          "crater_density must be ≥ 0"
        assert 0 <= self.masked_fraction <= 1,    "masked_fraction must be in [0, 1]"
        assert 0 < self.overlap_fraction <= 1,    "overlap_fraction must be in (0, 1]"
        assert self.tile_count >= 1,              "tile_count must be ≥ 1"
```

**Input/output contract:**
- Input: `PairRecord` + L1 `meta.json` (dict).
- Output: `FeatureVector` instance.
- Raised exceptions: `KeyError` if a required L1 meta field is missing (caller catches and
  falls back to benchmark mode — see CONFIGURATION.md §1 `on_feature_extraction_error`).

### 2.2 SelectorResult

```python
# src/selector/model.py

MATCHER_NAMES = ["sift", "rift2", "lightglue", "crater"]   # index → name mapping
ROUTING_REASONS = frozenset({
    "high_confidence",
    "low_confidence_fallback",
    "very_low_confidence_full_mode",
    "benchmark_mode",           # MSM disabled; all matchers run
    "model_unavailable",        # model file missing; fallback activated
    "feature_error",            # feature extraction failed; fallback activated
})

@dataclass
class SelectorResult:
    pair_id:              str
    selected_matcher:     str         # one of MATCHER_NAMES
    confidence:           float       # max(all_probs) after hard-rule masking; in [0, 1]
    fallback_matcher:     str         # 2nd-highest probability matcher
    all_probs:            dict        # {matcher_name: float}; sums to 1.0
    routing_reason:       str         # one of ROUTING_REASONS
    matchers_to_run:      list[str]   # the actual matchers S4 will run
    hard_rules_applied:   list[str]   # e.g. ["crater_density_gate", "gpu_gate"]
    selector_version:     str         # e.g. "msm_v1"
    feature_vector_hash:  str         # sha256 of FeatureVector.to_array().tobytes()

    def to_json(self) -> dict:
        return dataclasses.asdict(self)

    @classmethod
    def from_json(cls, d: dict) -> "SelectorResult":
        return cls(**d)
```

**Invariants (enforced by `MatcherSelector.route()`):**
- `sum(all_probs.values())` ∈ [0.999, 1.001].
- `selected_matcher == argmax(all_probs)`.
- `confidence == all_probs[selected_matcher]`.
- `routing_reason` ∈ `ROUTING_REASONS`.
- `matchers_to_run` is non-empty.
- If `routing_reason == "high_confidence"`: `len(matchers_to_run) == 1`.
- If `routing_reason == "low_confidence_fallback"`: `len(matchers_to_run) == 1` (the fallback).
- If `routing_reason in {"very_low_confidence_full_mode", "benchmark_mode"}`: all applicable
  matchers are in `matchers_to_run` (subject to hard gates).

### 2.3 MatcherSelector

```python
# src/selector/model.py

class MatcherSelector:
    """
    Lightweight wrapper around a trained LightGBM model.
    Loads once at pipeline startup; inference is < 1 ms per pair on CPU.
    """

    def __init__(self, model_path: str, stats_path: str, cfg: MSMConfig):
        """
        Args:
            model_path : path to pickled sklearn Pipeline (LightGBM inside)
            stats_path : path to JSON with training distribution stats
            cfg        : validated MSMConfig dataclass (from msm.yaml)
        Raises:
            FileNotFoundError if model_path does not exist AND cfg.enabled is True.
            If cfg.enabled is False, __init__ succeeds even if model_path is missing.
        """
        ...

    def route(self, pair_record: PairRecord, l1_meta: dict) -> SelectorResult:
        """
        Core routing method.

        Args:
            pair_record : PairRecord for the current pair
            l1_meta     : dict loaded from data/processed/<pair_id>/meta.json

        Returns:
            SelectorResult with all fields populated.

        Never raises:
            Any exception from feature extraction, model inference, or hard-rule
            application is caught internally. On exception, returns a SelectorResult
            with routing_reason = "feature_error" or "model_unavailable" and
            matchers_to_run = all applicable matchers.
        """
        ...

    def _extract_features(self, pair_record: PairRecord, l1_meta: dict) -> FeatureVector:
        """Delegates to src/selector/features.py::extract_features()."""
        ...

    def _apply_hard_rules(self, probs: np.ndarray, pair_record: PairRecord) -> np.ndarray:
        """
        Zeros and renormalizes probabilities according to hard rules.
        Returns renormalized probabilities (same shape, sums to 1.0).
        If all probabilities are zeroed (degenerate): returns uniform distribution
        over non-gated matchers (M0 always has prob > 0).
        """
        ...

    def _check_distribution_shift(self, fv: FeatureVector) -> list[str]:
        """
        Returns list of feature names that are outside training distribution ± 2σ.
        Used for warning logging only.
        """
        ...
```

**Thread safety:** `MatcherSelector` is read-only after `__init__`. The same instance may be
shared across parallel pair-processing workers without locking.

**Lifecycle:**
1. `benchmark.py` instantiates one `MatcherSelector` at startup (or skips if `msm.enabled=false`).
2. For each pair: `selector.route(pair_record, l1_meta)` → `SelectorResult`.
3. `SelectorResult` is written to `results/<pair_id>/selector.json`.
4. `benchmark.py` passes `SelectorResult.matchers_to_run` to the matcher registry loop.

### 2.4 MSMConfig

```python
# src/selector/config.py

from pydantic import BaseModel, validator

class HardRuleConfig(BaseModel):
    enabled:              bool = True
    tau_c:                float = 5.0       # for crater_density_gate
    check_at_startup:     bool = True       # for gpu_gate

class FallbackConfig(BaseModel):
    on_model_load_error:        str = "benchmark_mode"
    on_feature_extraction_error:str = "benchmark_mode"
    on_s4_gate_failure:         str = "sift"
    log_all_fallback_events:    bool = True

class LoggingConfig(BaseModel):
    write_selector_json:         bool = True
    write_feature_vector_json:   bool = True
    warn_on_out_of_range_features:bool = True

class TrainingConfig(BaseModel):
    min_labeled_pairs:  int = 50
    composite_score_weights: dict = {
        "rmse_norm_inv": 0.50,
        "inlier_ratio": 0.25,
        "spatial_coverage": 0.25,
    }
    lightgbm_params:    dict = { ... }   # defaults mirror msm.yaml
    cross_validation:   dict = { ... }

class MSMConfig(BaseModel):
    enabled:          bool = False
    model_path:       str  = "models/msm_v1.pkl"
    model_stats_path: str  = "models/msm_v1_stats.json"
    model_version:    str  = "msm_v1"
    tau_high:         float = 0.65
    tau_low:          float = 0.40
    hard_rules:       HardRuleConfig = HardRuleConfig()
    fallback:         FallbackConfig = FallbackConfig()
    logging:          LoggingConfig  = LoggingConfig()
    training:         TrainingConfig = TrainingConfig()

    @validator("tau_high")
    def tau_high_gt_tau_low(cls, v, values):
        if "tau_low" in values and v <= values["tau_low"]:
            raise ValueError("tau_high must be > tau_low")
        return v

    @validator("tau_low")
    def tau_low_ge_minimum(cls, v):
        if v < 0.25:
            raise ValueError("tau_low must be >= 0.25")
        return v
```

---

## 3. File artifacts produced by L1.5

| File | Written by | Read by | Format |
|---|---|---|---|
| `results/<pair_id>/selector.json` | `MatcherSelector.route()` | `benchmark.py`, audit scripts | JSON (SelectorResult) |
| `results/<pair_id>/msm_features.json` | `MatcherSelector.route()` | `train_msm.py`, audit scripts | JSON (FeatureVector.to_dict()) |
| `results/msm_fallback.jsonl` | `benchmark.py` (on S4 gate failure) | Human review, audit | JSONL (one event per line) |
| `results/msm_benchmark.csv` | `src/evaluation/msm_eval.py` | Human review, VALIDATION.md §2 | CSV |
| `models/msm_v1.pkl` | `train_msm.py` | `MatcherSelector.__init__()` | Pickle (sklearn Pipeline) |
| `models/msm_v1_stats.json` | `train_msm.py` | `MatcherSelector._check_distribution_shift()` | JSON |

---

## 4. Integration points (where L1.5 touches existing code)

| Existing file | Change type | Description |
|---|---|---|
| `src/ingest/records.py` | Add field | `PairRecord.msm_split: str` (defaults to `split`) |
| `scripts/preprocess.py` | Add output | MSM feature fields written to `meta.json` |
| `scripts/benchmark.py` | Add routing | Reads `selector.json`; passes `matchers_to_run` to registry loop |
| `scripts/build_pairs.py` | Add field | Writes `msm_split` to manifest |
| `src/evaluation/leakage_audit.py` | Add check | `--check-msm` flag verifies MSM train/test geo-cell disjointness |
| `configs/matchers.yaml` | Add fields | `msm_safe_fallback` and `msm_matcher_order` keys |

All other files (`matching/`, `selection/`, `registration/`, `refinement/`, evaluation metrics)
are **untouched** by the MSM addition.

---

## 5. Versioning and backward compatibility

- `selector.json` includes `selector_version`. Stale `selector.json` files (from an older MSM
  version) are recognized by the version mismatch and ignored by `msm_eval.py` (they are not
  overwritten either — a re-run with `--force` is required to regenerate them).
- `FeatureVector.to_array()` must produce values in exactly `FEATURE_NAMES` order. Any change
  to `FEATURE_NAMES` (additions, reordering, removals) is a breaking change that requires
  incrementing `model_version` and retraining.
- `SelectorResult.matchers_to_run` is the single source of truth consumed by `benchmark.py`.
  Its format must remain a `list[str]` of valid matcher registry keys.
