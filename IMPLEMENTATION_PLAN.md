# SIH26166 — IMPLEMENTATION PLAN v2.0

Phased implementation guide for coding agents. Read ARCHITECTURE.md, INTERFACES.md, and CONFIGURATION.md first. This document tells you what order to build things and exactly what each file must contain.

**Read DECISIONS.md before changing any algorithm. Read VALIDATION.md before running any evaluation.**

---

## Phase 0 — Environment and Project Scaffold

**Goal:** Working conda environment + repository skeleton + pilot data downloaded.

### 0.1 Environment

```bash
conda create -n asp -c conda-forge -c usgs-astrogeology ames-stereo-pipeline
conda activate asp
pip install pyyaml tqdm rasterio shapely pygeodesy lightglue kornia numpy scipy opencv-python-headless
# verify ASP version
stereo_gui --version  # must be >= 3.7.0
```

### 0.2 ISIS data

```bash
export ISISROOT=$CONDA_PREFIX
export ISISDATA=$HOME/projects/isisdata
export ALESPICEROOT=$ISISDATA
downloadIsisData chandrayaan2 $ISISDATA --exclude="kernels/ck/**"
# fetch ONLY the CK kernel window for your pilot strips (see PIPELINE.md S0)
```

### 0.3 Directory scaffold

```bash
mkdir -p SIH/{configs,data/{raw,calibrated,reference,pairs,processed,metadata/gt},src/{ingest,preprocessing,geometry,matching,selection,registration,refinement,evaluation},scripts,results/{pair_results},notebooks,app}
touch SIH/data/pairs/{manifest.jsonl,skipped.jsonl,failures.jsonl}
```

### 0.4 Pilot data

Download from PRADAN/CHMAP:
- 2 verified OHRC strips (original filenames unchanged)
- 1 TMC-2 ortho/DEM (ASP docs §8.15 set)
Download from Lunar ODE: matching NAC strip for each OHRC
Download WAC 643nm mosaic (GDAL crop tool or Moon Trek)

---

## Phase 1 — Data and Geometry Layer (F01–F03)

**Files to create:**

### src/ingest/label_parser.py
```python
from pathlib import Path
import json, subprocess
from dataclasses import dataclass, asdict
from typing import Optional

@dataclass
class ProductMeta:
    product_id: str
    cub_path: str
    gsd_m: float
    solar_incidence_deg: float
    solar_azimuth_deg: float
    sensor: str           # "OHRC" | "TMC2" | "IIRS" | "NAC" | "WAC"
    utc: str
    footprint_ll: list    # [[lon,lat], x4]
    footprint_shape: list # [height_px, width_px]

def parse_pds4_label(xml_path: Path) -> ProductMeta:
    """Parse OHRC/TMC-2 .xml PDS4 label.
    Extract: product_id, GSD, solar angles, UTC, corner lat/lon.
    Return ProductMeta."""
    ...

def run_isisimport(img_path: Path, out_dir: Path) -> Path:
    """Run isisimport on the .img file.
    Returns path to the produced .cub file.
    NEVER rename img_path -- isisimport depends on original ISRO filename."""
    ...

def run_spiceinit(cub_path: Path) -> bool:
    """Run spiceinit on the .cub file. Return True on success."""
    ...
```

### src/ingest/reference.py
```python
from pathlib import Path

def query_ode_nac(footprint_ll: list, padding_m: float = 3000) -> Optional[Path]:
    """Query Lunar ODE bbox endpoint for overlapping NAC strips.
    Returns downloaded crop path or None."""
    ...

def crop_wac_mosaic(mosaic_path: Path, bbox_ll: list) -> Path:
    """GDAL crop of WAC 643nm mosaic to bounding box.
    bbox_ll = [lon_min, lat_min, lon_max, lat_max].
    Returns cropped GeoTIFF path."""
    ...

def pad_bbox(footprint_ll: list, sigma_m: float, k: float = 3.0) -> list:
    """Expand footprint bbox by k * sigma_m in all directions.
    Returns [lon_min, lat_min, lon_max, lat_max]."""
    ...
```

### scripts/ingest.py
Entry point for Phase 1. Reads raw dir, calls label_parser + isisimport + spiceinit + reference query, writes products.jsonl. Handles failures with logging to failures.jsonl.

### scripts/build_pairs.py
Reads products.jsonl, calls reference.py, computes terrain_class and crater_density (initial estimate from WAC DEM if available, else None), assigns geo_cell and split, writes manifest.jsonl.

**Phase 1 complete when:** `manifest.jsonl` has 3+ entries with valid footprints and reference crops.

---

## Phase 2 — Preprocessing (F04–F08)

**Files to create:**

### src/preprocessing/masks.py
```python
import numpy as np

def shadow_mask(image: np.ndarray,
                solar_incidence_deg: float,
                incidence_threshold: float = 80.0,
                local_variance_window: int = 15,
                flat_variance_threshold: float = 10.0) -> np.ndarray:
    """Compute boolean validity mask.
    True = valid pixel; False = shadow / flat / invalid.
    Returns mask of same shape as image (single channel)."""
    ...

def check_mask_fraction(mask: np.ndarray,
                         min_pct: float = 5.0,
                         max_pct: float = 30.0) -> tuple:
    """Returns (fraction_masked, in_range_bool)."""
    ...
```

### src/preprocessing/normalize.py
```python
def percentile_clip(image: np.ndarray, lo: float = 2, hi: float = 98) -> np.ndarray: ...
def stat_transfer(src: np.ndarray, ref: np.ndarray) -> np.ndarray:
    """Transfer mean and std of ref to src. Return normalized src."""
    ...
```

### src/preprocessing/branches.py
```python
def apply_ohrc_nac(image: np.ndarray, config: dict) -> np.ndarray:
    """CLAHE + optional inversion + morphological dilation + PCA."""
    ...

def apply_tmc_wac(image: np.ndarray, ref: np.ndarray, config: dict) -> np.ndarray:
    """Histogram match + CLAHE. EXPERIMENTAL -- A/B test this."""
    ...

def apply_minimal(image: np.ndarray, config: dict) -> np.ndarray:
    """Only percentile clip. Used for M2/M3 (learned matchers)."""
    ...
```

### src/preprocessing/resample.py
```python
def reconcile_gsd(src: np.ndarray, src_gsd: float,
                  ref_gsd: float, solar_incidence_deg: float,
                  low_angle_threshold: float = 45.0) -> np.ndarray:
    """Resample src to match ref GSD.
    Use bilinear if solar_incidence >= threshold, bicubic otherwise.
    Returns resampled image."""
    ...
```

### src/preprocessing/tiling.py
```python
def tile_image(image: np.ndarray, tile_size: int = 512,
               overlap_px: int = 64,
               min_fraction: float = 0.5) -> list:
    """Return list of (tile_array, (row_offset, col_offset)) tuples.
    Discard tiles smaller than min_fraction * tile_size."""
    ...

def write_tile_geojson(tiles: list, pair_id: str, out_path: str): ...
```

### scripts/preprocess.py
Entry point. Reads manifest.jsonl, runs full L1 pipeline per pair, writes data/processed/<pair_id>/{src.tif, ref.tif, valid_mask.png, tiles.geojson, meta.json}.

**Phase 2 complete when:** `data/processed/<pair_id>/` exists for 3 pilot pairs; `meta.json` has provenance log; mask fraction is reasonable.

---

## Phase 3 — Correspondence Engine and Uniformity (F09–F14)

### src/matching/base.py (must be done first)
See INTERFACES.md §9 for the exact ABC definition. Implement it exactly.

### src/selection/anms.py
```python
def anms_ssc(keypoints: list, num_points: int,
             image_shape: tuple) -> list:
    """Suppression via Square Covering (SSC) variant of ANMS.
    keypoints: list of cv2.KeyPoint, sorted by response descending.
    Returns filtered list of at most num_points keypoints.
    No two returned keypoints within the suppression radius.
    Reference: Bailo et al., Pattern Recognition Letters 2018."""
    ...
```

### src/matching/sift.py (M0)
```python
from .base import BaseMatcher, MatchResult
from ..selection.anms import anms_ssc

class SIFTMatcher(BaseMatcher):
    matcher_id = "sift"
    requires_gpu = False

    def match(self, src, ref, valid_mask_src=None, valid_mask_ref=None) -> MatchResult:
        # detect -> ANMS SSC -> describe -> brute-force L2 match -> Lowe ratio 0.75
        # return MatchResult
        ...
```

### src/matching/rift.py (M1)
```python
class RIFT2Matcher(BaseMatcher):
    """RIFT2 + multi-octave log-Gabor scale-space extension.

    Algorithm:
    1. Log-Gabor filter bank (Ns scales, No orientations) over multiple octaves
    2. Phase Congruency (PC) map -> minimum/maximum moment keypoints (corners + edges)
    3. ANMS SSC on PC keypoints
    4. MIM descriptor: argmax orientation channel index per pixel -> 6x6xNo histogram
    5. Rotation invariance: No candidate MIMs per target keypoint, match all
    6. NCC-based matching

    IMPORTANT: scale_space_octaves is our addition (closes RIFT's scale gap).
    Calibrate pc_threshold on 2-3 pilot pairs before full run.
    Expect runtime 60-120s per tile on CPU -- flag if exceeded.
    """
    matcher_id = "rift2"
    requires_gpu = False

    def _log_gabor_bank(self, image, n_scales, n_orientations, octave): ...
    def _phase_congruency(self, responses): ...
    def _mim_descriptor(self, pc_map, responses, kpt): ...
    def match(self, src, ref, valid_mask_src=None, valid_mask_ref=None) -> MatchResult: ...
```

### src/matching/lightglue.py (M2)
```python
from lightglue import SuperPoint, LightGlue, match_pair
from .base import BaseMatcher, MatchResult
from ..registration.checks import f2_checks

class LightGlueMatcher(BaseMatcher):
    matcher_id = "lightglue"
    requires_gpu = True

    def match(self, src, ref, valid_mask_src=None, valid_mask_ref=None) -> MatchResult:
        # run SuperPoint + LightGlue
        # MANDATORY: f2_checks(matches) before returning -- never skip
        # store confidence per match from LightGlue output
        ...
```

### src/matching/crater.py (M3)
```python
class CraterMatcher(BaseMatcher):
    """CNSFM-style crater-geometry matcher.

    Gate: BOTH images must have crater_density >= tau_c.
    If gate fails: return empty MatchResult with gate_skip=True.

    Algorithm:
    1. YOLOv9 crater detection (transfer-learned from DeepMoon / TMC-2 crater paper)
    2. CNSF construction: for each detected crater, record center + radius + neighborhood topology
    3. Similarity-invariant topology matching across image pair
    4. MCR structural outlier removal
    5. Return crater centers as match points

    Implementation note: YOLOv9 weights must be downloaded and documented.
    If YOLOv9 unavailable, crater branch can be replaced with a classical circle-detection
    fallback (HoughCircles) -- document this in results as 'crater_hough', not 'crater'.
    """
    matcher_id = "crater"
    requires_gpu = True

    def _detect_craters(self, image): ...
    def _build_cnsf(self, craters): ...
    def _topology_match(self, cnsf_src, cnsf_ref): ...
    def match(self, src, ref, valid_mask_src=None, valid_mask_ref=None) -> MatchResult: ...
```

### src/selection/spatial.py
```python
def confidence_filter(matches, threshold): ...
def grid_cap(matches, n=8, cap=5, image_shape=None): ...
def coverage_greedy(matches, budget=250, min_coverage=0.60): ...
def one_to_one(matches): ...
def selection_stats(before, after, image_shape): ...
```

### scripts/benchmark.py
Entry point. Reads manifest.jsonl, loops over pairs and matchers, runs L2+L3 per (pair, matcher), writes matches_raw.json and matches_selected.json + selection_stats.json. Handles GPU lock for M2.

**Phase 3 complete when:** matches_selected.json exists for at least M0 (SIFT) and M2 (LightGlue) on 3 pilot pairs.

---

## Phase 4 — Verification, Refinement, Products, Evaluation (F15–F25)

### src/registration/checks.py (F15)
```python
def f2_checks(matches, src_shape, ref_shape, buffer_px=10):
    """Mandatory. Remove out-of-bounds and duplicate matches.
    Called before ANY RANSAC/DEGENSAC step.
    Returns filtered matches + counts of removed."""
    ...
```

### src/registration/ladder.py (F16, F17)
```python
def degensac_verify(matches, model, threshold_px, max_iter=10000, confidence=0.99999): ...
def model_ladder(matches, src_shape, ref_shape, config): ...
def tilewise_models(matches, src_shape, ref_shape, config): ...
```

Note: DEGENSAC is available via pydegensac package (`pip install pydegensac`) or via OpenCV's USAC_MAGSAC. MAGSAC++ is an acceptable alternative.

```bash
pip install pydegensac
```

### src/registration/declustering.py (F18)
```python
def decluster(inliers, min_spacing_px=20, image_shape=None): ...
def zscore_filter(inliers, threshold=3.0, min_gcps=20): ...
```

### src/refinement/local.py (F19)
```python
def refine_match(src_full, ref_full, src_xy, ref_xy,
                 window_px=32, method='ncc',
                 apodization='tukey', pyramid_levels=3,
                 sharpness_threshold=0.15):
    """
    APODIZATION MUST be 'tukey' or 'gaussian'. NEVER 'blackman'.
    Returns (refined_ref_xy, delta_xy, sharpness, success_bool).
    """
    ...

def paraboloid_peak(corr_surface):
    """Fit 2D paraboloid to peak of NCC/POC surface.
    Returns (dx, dy, sharpness) as sub-pixel offset."""
    ...
```

### scripts/register.py (F20, F21)
Entry point. Reads geometry.json + matches_refined.json, warps source onto reference grid, writes GeoTIFF, CSV, GCP, and QC images.

### src/evaluation/ (F22, F23)
- metrics.py: rmse, pct_lt_1px, pct_lt_0p5px, medae, spatial_coverage, grid_density_std
- aggregate.py: reads all pair JSONs, aggregates by stratum, writes leaderboard.csv
- leakage_audit.py: verifies no geo_cell overlaps between train and test
- arbitration.py: determines winning matcher per pair, writes arbitration.log

**Phase 4 complete when:**
- leaderboard.csv has at least M0 results on 3 pilot pairs
- leakage audit passes
- One QC checkerboard image looks correctly aligned by eye in QGIS

---

## Implementation Constraints

**Coordinate convention:** ALWAYS use (col, row) = (x, y) for pixel coordinates. Add an assertion at the top of every function that touches coordinates:
```python
assert src_xy.shape[-1] == 2, "Expected (N,2) array: (col, row)"
```

**Seed:** Set before all random operations:
```python
import numpy as np, random
np.random.seed(config['global']['seed'])
random.seed(config['global']['seed'])
```

**Error handling:** ALL failures must be caught and written to failures.jsonl with stage, reason, and fallback_taken. Never let a single pair crash the full pipeline.

**Provenance:** Every file written must be accompanied by meta.json or embedded metadata with: config_hash, code_commit, matcher_params_hash, created_at. Use:
```python
import hashlib, json
def hash_config(cfg): return hashlib.md5(json.dumps(cfg, sort_keys=True).encode()).hexdigest()
```

**Never:** Rename ISRO files; apply heavy preprocessing to M2/M3; use Blackman apodization; skip F2 checks; use full 200 GB CK kernel set; publish leaderboard numbers before leakage audit passes.
