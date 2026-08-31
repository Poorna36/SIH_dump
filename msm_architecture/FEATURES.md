# SIH26166 — FEATURES (Matcher Selection Model Input Features, v1.0)

Defines every input feature consumed by the Matcher Selection Model (MSM / L1.5). All features
are extracted from the `PairRecord` (already in `data/pairs/manifest.jsonl`) and the L1 `meta.json`
(written by `scripts/preprocess.py`). No additional image I/O or processing is required at
inference time.

Related docs: ARCHITECTURE.md §L1.5, INTERFACES.md §FeatureVector, CONFIGURATION.md §msm.yaml,
VALIDATION.md §2.

---

## 1. Feature table

| # | Name | Type | Source | Range / Categories | Notes |
|---|---|---|---|---|---|
| 1 | `sensor_pair_enc` | int (encoded) | PairRecord | 0=OHRC→NAC, 1=TMC→WAC, 2=IIRS→WAC | One-hot encoding passed to LightGBM as categorical |
| 2 | `gsd_ratio` | float | PairRecord | (0, 1] | src_gsd / ref_gsd; always ≤ 1 by convention (src = finer) |
| 3 | `latitude_abs` | float | PairRecord bbox centroid | [0°, 90°] | abs(mean(lat_min, lat_max)) |
| 4 | `delta_solar_azimuth` | float | PairRecord illum | [0°, 180°] | abs(src_azimuth − ref_azimuth); clamped to [0, 180] |
| 5 | `terrain_class_enc` | int (encoded) | PairRecord | 0=highland, 1=maria, 2=polar, 3=mixed | Categorical |
| 6 | `crater_density` | float | PairRecord | [0, ∞) craters/Mpx | Log1p-transformed before model input |
| 7 | `masked_fraction` | float | L1 meta.json | [0.0, 1.0] | Fraction of pixels flagged by shadow/validity mask |
| 8 | `overlap_fraction` | float | PairRecord | (0.0, 1.0] | Fraction of source footprint covered by reference crop |
| 9 | `src_texture_contrast` | float | L1 meta.json | [0, ∞) DN | Mean local σ in 8×8 pixel windows over source patch |
| 10 | `ref_texture_contrast` | float | L1 meta.json | [0, ∞) DN | Same for reference patch |
| 11 | `src_mean_gradient` | float | L1 meta.json | [0, ∞) DN/px | Mean Sobel gradient magnitude of source patch after normalization |
| 12 | `ref_mean_gradient` | float | L1 meta.json | [0, ∞) DN/px | Same for reference patch |
| 13 | `tile_count` | int | L1 meta.json | [1, ~100] | Number of valid (non-discarded) tiles after GSD reconciliation |

**Total: 13 features.** LightGBM handles mixed categorical/continuous natively; no manual
one-hot expansion is needed — `sensor_pair_enc` and `terrain_class_enc` are declared as
`categorical_feature` in the LightGBM config.

---

## 2. Feature extraction (implementation)

### 2.1 Features from PairRecord (zero additional I/O)

These are read directly from the JSON manifest at inference time:

```python
# src/selector/features.py

SENSOR_PAIR_MAP = {"OHRC_NAC": 0, "TMC_WAC": 1, "IIRS_WAC": 2}
TERRAIN_MAP     = {"highland": 0, "maria": 1, "polar": 2, "mixed": 3}

def extract_from_pair_record(pr: PairRecord) -> dict:
    lat_center = (pr.bbox_pad_deg[1] + pr.bbox_pad_deg[3]) / 2.0
    key = f"{pr.source.sensor}_{pr.reference.sensor}"
    return {
        "sensor_pair_enc":      SENSOR_PAIR_MAP.get(key, -1),
        "gsd_ratio":            float(pr.gsd_ratio),
        "latitude_abs":         abs(float(lat_center)),
        "delta_solar_azimuth":  min(abs(pr.illum.delta_azimuth), 180.0),
        "terrain_class_enc":    TERRAIN_MAP.get(pr.terrain_class, 3),
        "crater_density":       math.log1p(float(pr.crater_density)),
        "overlap_fraction":     float(pr.overlap_fraction),
    }
```

### 2.2 Features from L1 meta.json (computed during preprocessing, stored on disk)

These are written once by `scripts/preprocess.py` and read back at inference time:

```python
# scripts/preprocess.py (addition to existing output)

import cv2, numpy as np

def compute_msm_features(src_norm: np.ndarray, ref_norm: np.ndarray,
                          valid_mask: np.ndarray, tile_count: int) -> dict:
    """
    src_norm, ref_norm: float32 arrays [0,1], normalized images (post-L1 processing).
    valid_mask: bool array, True = valid pixel.
    """
    def texture_contrast(img):
        # Mean local standard deviation in 8×8 non-overlapping windows
        h, w = img.shape[:2]
        stds = []
        for i in range(0, h - 8, 8):
            for j in range(0, w - 8, 8):
                patch = img[i:i+8, j:j+8]
                stds.append(patch.std())
        return float(np.mean(stds)) if stds else 0.0

    def mean_gradient(img):
        # Mean Sobel gradient magnitude (L1-norm of Gx, Gy)
        gx = cv2.Sobel(img, cv2.CV_32F, 1, 0, ksize=3)
        gy = cv2.Sobel(img, cv2.CV_32F, 0, 1, ksize=3)
        return float(np.mean(np.sqrt(gx**2 + gy**2)))

    return {
        "masked_fraction":      float((~valid_mask).mean()),
        "src_texture_contrast": texture_contrast(src_norm),
        "ref_texture_contrast": texture_contrast(ref_norm),
        "src_mean_gradient":    mean_gradient(src_norm),
        "ref_mean_gradient":    mean_gradient(ref_norm),
        "tile_count":           int(tile_count),
    }
```

### 2.3 Full FeatureVector assembly

```python
# src/selector/features.py

import numpy as np

FEATURE_NAMES = [
    "sensor_pair_enc", "gsd_ratio", "latitude_abs", "delta_solar_azimuth",
    "terrain_class_enc", "crater_density", "masked_fraction", "overlap_fraction",
    "src_texture_contrast", "ref_texture_contrast",
    "src_mean_gradient", "ref_mean_gradient", "tile_count",
]

CATEGORICAL_FEATURES = ["sensor_pair_enc", "terrain_class_enc"]

@dataclass
class FeatureVector:
    sensor_pair_enc:      int
    gsd_ratio:            float
    latitude_abs:         float
    delta_solar_azimuth:  float
    terrain_class_enc:    int
    crater_density:       float   # already log1p-transformed
    masked_fraction:      float
    overlap_fraction:     float
    src_texture_contrast: float
    ref_texture_contrast: float
    src_mean_gradient:    float
    ref_mean_gradient:    float
    tile_count:           int

    def to_array(self) -> np.ndarray:
        return np.array([getattr(self, f) for f in FEATURE_NAMES], dtype=np.float32)

    def to_dict(self) -> dict:
        return {f: getattr(self, f) for f in FEATURE_NAMES}


def extract_features(pair_record: PairRecord, l1_meta: dict) -> FeatureVector:
    pr_feats  = extract_from_pair_record(pair_record)
    l1_feats  = {k: l1_meta[k] for k in
                 ["masked_fraction", "src_texture_contrast", "ref_texture_contrast",
                  "src_mean_gradient", "ref_mean_gradient", "tile_count"]}
    return FeatureVector(**{**pr_feats, **l1_feats})
```

---

## 3. Feature rationale (why each feature predicts matcher performance)

| Feature | Predicts |
|---|---|
| `sensor_pair_enc` | Strong prior: OHRC→NAC benefits most from learned M2; IIRS→WAC requires separate track; TMC→WAC is intermediate. |
| `gsd_ratio` | Low ratio (large scale gap) hurts SIFT (scale invariance limit) but M1 RIFT's scale-space extension handles it; M2 LightGlue also handles it. |
| `latitude_abs` | >55° = SIFT fails (17.3% SR documented); M3 Crater excels at polar; M1 RIFT robust across latitudes. |
| `delta_solar_azimuth` | >90° Δ-azimuth = strong illumination reversal; M1 (PC-based) and M2 (attention-based) both handle this; M0 SIFT fails. |
| `terrain_class_enc` | Maria = crater-sparse → M3 disabled; Highland/polar = crater-rich → M3 viable; M2 generalizes. |
| `crater_density` | Direct gate predictor for M3 (F14). Also indicates textural richness for SIFT/RIFT. |
| `masked_fraction` | High masked fraction → fewer valid features → M2's global attention handles sparse valid regions better than local descriptors. |
| `overlap_fraction` | Low overlap → fewer candidate matches → M2's confidence scoring more reliable; SIFT needs high overlap. |
| `src_texture_contrast` | Low contrast → SIFT/RIFT suffer (phase congruency helps in low-contrast); M2 degrades gracefully. |
| `ref_texture_contrast` | Same. |
| `src_mean_gradient` | Low gradient → flat mare/textureless; all classical matchers struggle; M2 > M0 > M1. |
| `ref_mean_gradient` | Same. |
| `tile_count` | Many tiles → large scene, possible heterogeneity → M3 Crater or tiled SIFT more reliable than a single global M2 pass. |

---

## 4. Feature normalization

LightGBM is tree-based and does **not** require feature normalization for training or inference.
However, for interpretability and to catch distribution-shift bugs at inference time:

- All continuous features are range-checked against the bounds in the table in §1.
- A warning is logged (not an error) if any feature at inference time falls outside
  `[train_min - 2σ, train_max + 2σ]` of the training distribution.
- `crater_density` is log1p-transformed before training; the same transform is applied at inference.
- The training-distribution stats are stored in `models/msm_v1_stats.json` alongside the model.

---

## 5. Feature importance audit (required before MSM activation)

After training, run:

```bash
python scripts/train_msm.py \
  --leaderboard results/leaderboard.csv \
  --manifest data/pairs/manifest.jsonl \
  --processed data/processed \
  --split msm_split --split-value train \
  --out models/msm_v1.pkl \
  --plot-importance results/figures/msm_feature_importance.png
```

**Acceptance criterion**: all five top-importance features must have non-zero LightGBM gain.
If any single feature dominates (>50% gain), inspect for leakage (e.g. `crater_density`
perfectly predicting M3 in a dataset where M3 was gated correctly — this is expected and correct,
not leakage).

---

## 6. Features explicitly excluded (and why)

| Excluded feature | Reason |
|---|---|
| RMSE of any matcher | Only available after matching — would not be available at selection time. |
| Inlier count / ratio | Same — post-matching. |
| Descriptor statistics (e.g., SIFT keypoint count) | Not available before matching; would require running the detector, defeating the purpose. |
| Image histogram percentiles | Redundant with `texture_contrast` and `mean_gradient`; adds I/O. |
| GPU availability at inference time | Handled by hard-rule masking (not a learned feature). |
| Product ID / timestamp | Would encode orbit-specific correlations that do not generalize — leakage risk. |
| Geo-cell id | Directly encodes the train/test split — leakage. |
| `split` / `msm_split` field | Same. |

---

## 7. Backward compatibility

These features are added to `L1 meta.json` as new fields. Existing code that reads `meta.json`
but does not reference these fields is unaffected. Pairs processed before this update will have
`meta.json` files without the MSM fields; re-running `scripts/preprocess.py --force` on those
pairs will add the missing fields. Until re-processed, those pairs cannot be used for MSM training
or inference (they are silently skipped with a warning, and the MSM falls back to benchmark mode
for those pairs).
