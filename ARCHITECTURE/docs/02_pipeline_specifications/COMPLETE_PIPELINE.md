# COMPLETE PIPELINE — SIH 26166
## Consolidated System Architecture
### All Research + AI Suggestions Integrated

---

# CROSS-REFERENCE: AI SUGGESTIONS VERIFICATION

Before the pipeline: confirmation that every recommendation from the 3 AI suggestion files has been addressed.

### [SIH26166_supplementary_research(2).md](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/SIH26166_supplementary_research%282%29.md) — 8-item suggested stack

| Their Recommendation | Our Decision | Justification |
|---|---|---|
| 1. ISIS3 + ASP for calibration | ✅ **Adopted** — Stage 2 | Community standard. No alternative for Chandrayaan-2 PDS4 ingestion. |
| 2. LNIFT as primary correspondence | ⚠️ **Overridden** — LightGlue chosen instead | LNIFT has zero lunar polar/cross-modal evidence. LightGlue succeeded on ALL 6 test conditions on our exact sensors [Traditional_vs_DL]. LNIFT positioned as comparison baseline. |
| 3. RIFT/RIFT2/OS-SIFT/CFOG baselines | ✅ **Adopted** — evaluation comparisons | RIFT2 included as Architecture B component and ablation baseline. |
| 4. ANMS for uniform distribution | ✅ **Adopted** — Stage 4 (ESSENTIAL) | SSC variant chosen. Grid-based top-k as fallback. |
| 5. RANSAC for outlier rejection | ✅ **Upgraded** — DEGENSAC instead of RANSAC | Standard RANSAC risks degeneracy on flat lunar mare. DEGENSAC adds degeneracy awareness [LoFTR_IMC21]. |
| 6. ASP subpixel-mode 9 for refinement | ✅ **Adapted** — Phase-correlation paraboloid fitting | Same principle (analytical sub-pixel peak fitting). Paraboloid fitting demonstrated RMSE < 0.2 px [HybridPhaseCorrelation] — stronger evidence than ASP's internal method. |
| 7. RMSE + inlier ratio + spatial distribution + timing | ✅ **Adopted + extended** — Stage 7 metrics | Added MedAE, %<1px, %<0.5px from [HybridPhaseCorrelation]. |
| 8. IIRS as separate module | ✅ **Adopted** — Sensor-pair profile: transitive registration | IIRS↔WAC treated as separate path. OHRC↔IIRS (320×) uses transitive chain. |

### [Additional_Resources_SIH26166.md](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/Additional_Resources_SIH26166.md) — 13 items

| Item | Status | Where Used |
|---|---|---|
| OHRC DEM papers (ASP + ISIS) | ✅ Adopted | Training data Source 2 (ASP-derived ground truth) |
| OHRC Shape-from-Shading paper | ✅ Noted | Future: photometric refinement (not core pipeline) |
| Synthetic lunar image tools | ✅ Adopted | Training data Source 4 (physically-based rendering) |
| RIFT | ✅ Adopted | Full extraction read. PC keypoints → Stage 3A supplement. RIFT2 → Architecture B. |
| MatchAnything | ✅ Tracked | Excluded Methods: "HIGH PRIORITY for future investigation" |
| RoMa | ✅ Tracked | Excluded Methods: "could replace LightGlue if cross-modal performance proves superior" |
| CM-Bench | ✅ Noted | Evaluation plan: template for cross-modal evaluation protocol |
| ASP + ISIS | ✅ Adopted | Stage 2 calibration (ESSENTIAL) |
| hloc toolbox | ✅ Adopted | Stage 3A implementation base |
| LightGlue | ✅ Adopted | Stage 3A core matcher (ESSENTIAL) |
| DeepMoon | ✅ Adopted | Stage 3B crater detector option |
| TMC-2 crater U-Net | ✅ Adopted | Stage 3B crater detector option |
| Lower priority (PromptMID, SEN1-2, etc.) | ✅ Excluded | Heavier, less proven than adopted methods |

### [SIH26166_reference_mapping.md](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/SIH26166_reference_mapping.md) — 5 concepts

| Concept | Status | Where Used |
|---|---|---|
| LOLA as geodetic control / trusted frame | ✅ Adopted | Architecture context — WAC is reference because it's tied to LOLA |
| WAC as standard reference basemap | ✅ Adopted | Stage 1 reference selection |
| NAC for high-res reference strips | ✅ Adopted | OHRC↔NAC sensor-pair profile |
| SELENE as supplementary reference | ✅ Adopted | Alternative reference option |
| Metadata → padded patch → matching workflow | ✅ Adopted | Stage 1 (ESSENTIAL) — the entire coarse-narrowing design |

> [!NOTE]
> **No architecture updates required.** All AI suggestions have been incorporated, adapted, or explicitly overridden with justification.

---

# COMPLETE SYSTEM PIPELINE

## Full Flow Diagram

```
╔══════════════════════════════════════════════════════════════════════╗
║                         RAW INPUTS                                  ║
║                                                                      ║
║  Source: Chandrayaan-2 PDS4 product (OHRC / TMC-2 / IIRS)          ║
║  Reference: LRO WAC global mosaic (75-100m, LOLA-tied geodetic)    ║
║             OR LRO NAC strip (0.5-2m, for OHRC high-res matching)  ║
║             OR SELENE TC (10m, supplementary)                       ║
║  Metadata: Corner lat/lon, solar incidence/azimuth, sensor ID,     ║
║            spatial resolution                                        ║
╚══════════════════════════════╦═══════════════════════════════════════╝
                               ║
                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 1: METADATA-DRIVEN SPATIAL NARROWING                        ┃
┃  ─────────────────────────────────────────                         ┃
┃  Tools: pygeodesy, GDAL/rasterio, Moon Trek WMTS API               ┃
┃                                                                      ┃
┃  1. Extract corner coordinates from PDS4 metadata                    ┃
┃  2. Query reference dataset by bounding box                          ┃
┃     (local GDAL crop OR Moon Trek tile fetch)                        ┃
┃  3. Pad bbox 2-5× beyond pointing uncertainty (~500m-2km)           ┃
┃  4. Extract solar geometry:                                          ┃
┃     solar_angle_diff = |src_azimuth - ref_azimuth|                  ┃
┃  5. Compute resolution_ratio = src_res / ref_res                    ┃
┃  6. Classify difficulty:                                             ┃
┃     IF solar_angle_diff > 60° OR latitude > ±60° → EXTREME         ┃
┃     IF resolution_ratio > 10× → EXTREME_SCALE                      ┃
┃     ELSE → NORMAL                                                   ┃
┃                                                                      ┃
┃  Output: cropped source + reference patches,                        ┃
┃          difficulty_flag, resolution_ratio, solar_angle_diff         ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦═══════════════════════════════════━━━┛
                               ║
                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 2: PREPROCESSING & NORMALIZATION                             ┃
┃  ──────────────────────────────────────                             ┃
┃  Tools: ISIS3 (spiceinit, cam2map), OpenCV, numpy                   ┃
┃                                                                      ┃
┃  1. IF raw PDS4 → ISIS3 calibration:                                ┃
┃     isisimport → spiceinit → cam2map (radiometric + geometric)      ┃
┃                                                                      ┃
┃  2. Resolution harmonization:                                        ┃
┃     Resample lower-res image to match higher-res                     ┃
┃     IF solar_elevation < 30° → bilinear interpolation               ┃
┃     ELSE → bicubic interpolation                                    ┃
┃     [MoonMetaSync: bicubic amplifies shadow artifacts at low Sun]   ┃
┃                                                                      ┃
┃  3. Intensity normalization:                                         ┃
┃     Percentile clip (2nd/98th) → scale to [0,1]                    ┃
┃     Mean/std statistic transfer (source → reference)                ┃
┃     Convert to uint8                                                 ┃
┃     [SIFT-IIRS-WAC: validated on Chandrayaan-2 IIRS]               ┃
┃                                                                      ┃
┃  4. NO heavy enhancement (CLAHE/inversion/dilation/PCA)             ┃
┃     [Traditional_vs_DL: demonstrated insufficient for polar]        ┃
┃                                                                      ┃
┃  SPECIAL CASE — IIRS:                                               ┃
┃     Select most texture-rich band (visible range, ~0.8-1.0 µm)      ┃
┃     Treat as single-band panchromatic from here on                  ┃
┃                                                                      ┃
┃  Output: normalized uint8 image pair at matched resolution          ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦═══════════════════════════════════━━━┛
                               ║
                   ┌───────────╨───────────┐
                   │    ROUTING DECISION    │
                   │                        │
                   │  Check difficulty_flag │
                   │  + resolution_ratio    │
                   └──┬──────────┬────────┬┘
                      │          │        │
            NORMAL    │  EXTREME │        │ EXTREME_SCALE
            (most     │  ILLUM.  │        │ (OHRC↔IIRS
            cases)    │  (polar/ │        │  320× ratio)
                      │  high Δ) │        │
                      ▼          ▼        ▼
          ┌──────────────┐ ┌──────────┐ ┌──────────────────┐
          │  STAGE 3A    │ │ STAGE 3B │ │ TRANSITIVE PATH  │
          │  PRIMARY     │ │ EXTREME  │ │                  │
          │  MATCHER     │ │ ILLUM.   │ │ Match OHRC↔NAC   │
          │              │ │ PATH     │ │ Match IIRS↔WAC   │
          │ SuperPoint + │ │          │ │ Chain via known   │
          │ LightGlue    │ │ Crater   │ │ NAC↔WAC frame    │
          │              │ │ detection│ │                  │
          │              │ │ + CNSFM  │ │ [NASA_SubPixel:  │
          │              │ │          │ │  accuracy degrades│
          │              │ │          │ │  beyond 2-3×     │
          │              │ │          │ │  footprint ratio] │
          └──────┬───────┘ └────┬─────┘ └────────┬─────────┘
                 │              │                 │
                 │         ┌────┴────┐            │
                 │         │ enough  │            │
                 │    NO   │craters? │ YES        │
                 │◄────────┤ (≥10)   ├────┐       │
                 │         └─────────┘    │       │
                 │                        ▼       │
                 │              ┌──────────────┐  │
                 │              │ CNSFM Match  │  │
                 │              │ + MCR outlier│  │
                 │              │ removal      │  │
                 │              └──────┬───────┘  │
                 │                     │          │
                 │         ┌───────────┤          │
                 │         │ enough    │          │
                 │    NO   │ matches?  │ YES      │
                 │◄────────┤ (≥15)     ├────┐     │
                 │         └───────────┘    │     │
                 │                          │     │
                 ▼                          ▼     │
┌─────────────────────────────────────────────────┤
│                                                  │
│         ALL CORRESPONDENCES MERGE HERE           │
│         (with source_method tag per match)        │
│                                                  │
└─────────────────────┬────────────────────────────┘
                      │
                      ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 4: SPATIAL DISTRIBUTION ENFORCEMENT                          ┃
┃  ─────────────────────────────────────────                          ┃
┃  Tools: ANMS-Codes (github.com/BAILOOL/ANMS-Codes)                  ┃
┃                                                                      ┃
┃  1. Sort correspondences by confidence (descending)                  ┃
┃  2. Apply SSC (Suppression via Square Covering):                    ┃
┃     For each match: suppress if a stronger match exists within      ┃
┃     adaptive radius r                                                ┃
┃  3. Floor check: if remaining < 30, widen r and re-run              ┃
┃  4. Compute spatial coverage metric:                                 ┃
┃     Divide image into 8×8 grid                                       ┃
┃     coverage = 1 - (std_dev(counts) / mean(counts))                 ┃
┃                                                                      ┃
┃  FALLBACK: If ANMS problematic → grid-based top-k:                  ┃
┃     Divide into N×N grid, keep top-k by confidence per cell         ┃
┃  [Supplementary_Research §1: SSC is O(n), fastest ANMS variant]     ┃
┃                                                                      ┃
┃  Output: spatially uniform correspondence subset                    ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦═══════════════════════════════════━━━┛
                               ║
                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 5: ROBUST GEOMETRIC ESTIMATION                               ┃
┃  ────────────────────────────────────                               ┃
┃  Tools: pydegensac OR OpenCV USAC_DEGENSAC, numpy                   ┃
┃                                                                      ┃
┃  LAYER 1 — DEGENSAC:                                                ┃
┃    Model: similarity (4-param) → affine (6-param) if >30 inliers   ┃
┃    Threshold: 0.5 px reprojection error                              ┃
┃    Iterations: up to 10,000                                          ┃
┃    Confidence: 0.99999                                               ┃
┃    [LoFTR_IMC21: degeneracy-aware, guards flat lunar mare]          ┃
┃                                                                      ┃
┃  LAYER 2 — Global polynomial fit:                                   ┃
┃    Fit 2nd-order polynomial to all DEGENSAC inliers                 ┃
┃    via least squares                                                 ┃
┃    [SIFT-IIRS-WAC: decouples verification from final warp]         ┃
┃    [DESCA §P4: global affine fails on terrain relief]               ┃
┃                                                                      ┃
┃  LAYER 3 — Residual Z-score filter:                                 ┃
┃    IF inliers > 20:                                                  ┃
┃      Compute residuals from polynomial fit                           ┃
┃      Remove matches with |z-score| > 2.5                            ┃
┃    [SIFT-IIRS-WAC §P0: used operationally on Chandrayaan-2]        ┃
┃                                                                      ┃
┃  Output: inlier set + transform parameters + fit RMSE               ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦═══════════════════════════════════━━━┛
                               ║
                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 6: SUB-PIXEL REFINEMENT                                      ┃
┃  ─────────────────────────────                                      ┃
┃  Tools: numpy/scipy (FFT), skimage.registration                    ┃
┃                                                                      ┃
┃  For EACH inlier correspondence:                                     ┃
┃    1. Extract 64×64 px patch around match in both images             ┃
┃    2. Apply Gaussian window (avoid Blackman)                         ┃
┃       [HybridPhaseCorrelation §P4: Blackman suppresses too much]    ┃
┃    3. Compute phase-only cross-correlation:                          ┃
┃       F1 = FFT(patch1), F2 = FFT(patch2)                            ┃
┃       R = (F1 * conj(F2)) / |F1 * conj(F2)|                        ┃
┃       corr = IFFT(R)                                                 ┃
┃    4. Find integer peak location                                     ┃
┃    5. Fit 2D paraboloid to 3×3 neighborhood of peak:                ┃
┃       z = ax² + by² + cxy + dx + ey + f                             ┃
┃       Sub-pixel displacement = paraboloid vertex                     ┃
┃       [HybridPhaseCorrelation §P0: RMSE < 0.2 px demonstrated]     ┃
┃    6. REJECT if:                                                     ┃
┃       shift > 2.0 px (inconsistent with coarse match)               ┃
┃       OR peak correlation < 0.3 (unreliable)                        ┃
┃       → retain coarse position, flag is_refined = False             ┃
┃                                                                      ┃
┃  Output: sub-pixel-refined correspondence set                       ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦═══════════════════════════════════━━━┛
                               ║
                               ▼
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃  STAGE 7: VALIDATION & OUTPUT                                       ┃
┃  ────────────────────────────                                       ┃
┃  Tools: OpenCV (warpPerspective/warpAffine), numpy                  ┃
┃                                                                      ┃
┃  1. Composite confidence per match:                                  ┃
┃     conf = w1·matcher_score + w2·(1-ransac_residual/threshold)      ┃
┃            + w3·correlation_peak + w4·local_texture_variance        ┃
┃                                                                      ┃
┃  2. One-to-one enforcement:                                         ┃
┃     If multiple source → same reference: keep highest confidence    ┃
┃                                                                      ┃
┃  3. Warp source image using polynomial transform                    ┃
┃                                                                      ┃
┃  4. CRS reconciliation (metadata-based georeferencing alignment)    ┃
┃     [Traditional_vs_DL §P1: post-warp CRS re-alignment]            ┃
┃                                                                      ┃
┃  5. Compute quality metrics:                                        ┃
┃     RMSE = √(Σ(error²)/N)                                           ┃
┃     MedAE = median(|errors|)                                        ┃
┃     %<1px = count(error<1.0)/N × 100                                ┃
┃     %<0.5px = count(error<0.5)/N × 100                              ┃
┃     inlier_count = N                                                 ┃
┃     inlier_ratio = N / total_proposed_matches                       ┃
┃     spatial_coverage = 1-(std/mean) of grid density                 ┃
┃     [HybridPhaseCorrelation §P0: exact metric suite]                ┃
┃                                                                      ┃
┃  6. Success/failure determination:                                   ┃
┃     SUCCESS if RMSE < threshold AND inlier_count ≥ 15              ┃
┃     FAILURE otherwise → flag for manual review                      ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╦═══════════════════════════════════━━━┛
                               ║
                               ▼
╔══════════════════════════════════════════════════════════════════════╗
║                         FINAL OUTPUT                                ║
║                                                                      ║
║  1. Correspondence set:                                              ║
║     [(x_src, y_src, x_ref, y_ref, confidence, is_refined)] × N     ║
║                                                                      ║
║  2. Geometric transformation:                                        ║
║     Polynomial coefficients (2nd order, 12 params)                  ║
║                                                                      ║
║  3. Registered image:                                                ║
║     Source warped into reference CRS (GeoTIFF)                      ║
║                                                                      ║
║  4. Quality report:                                                  ║
║     {rmse, medae, pct_sub_1px, pct_sub_05px,                       ║
║      inlier_count, inlier_ratio, spatial_coverage,                  ║
║      runtime_seconds, success_flag,                                  ║
║      source_sensor, reference_sensor,                                ║
║      solar_angle_diff, difficulty_class}                             ║
╚══════════════════════════════════════════════════════════════════════╝
```

---

# STAGE 3 DETAIL: MATCHING PATHS

## Stage 3A: Primary Matcher (LightGlue)

```
┌─────────────────────────────────────────────────────────────┐
│                  STAGE 3A INTERNAL FLOW                       │
│                                                               │
│  source_norm ──┐                                              │
│                ├──→ SuperPoint ──→ keypoints_SP + desc_SP    │
│  reference_norm┘    (pretrained,                              │
│                      optional lunar                           │
│                      fine-tune)                               │
│                                                               │
│  source_norm ──┐    [EXPERIMENTAL]                            │
│                ├──→ Phase Congruency ──→ keypoints_PC        │
│  (log-Gabor     │   (No=6 orientations,    + desc_MIM        │
│   Ns=4 scales)  │    Ns=4 scales)                             │
│                                                               │
│  Merge: keypoints = keypoints_SP ∪ keypoints_PC              │
│         (deduplicate within 3px radius)                       │
│                                                               │
│  keypoints_src ──┐                                            │
│                  ├──→ LightGlue ──→ raw_matches              │
│  keypoints_ref ──┘    (Apache-2.0,    + confidence_scores    │
│                        via hloc)                              │
│                                                               │
│  Filter: keep matches with confidence > 0.2 (dustbin)        │
│  Bounds check: reject if x,y outside image domain             │
│  [HybridPhaseCorrelation §P4: LoFTR extrapolation warning]  │
│                                                               │
│  IF match_count < 15:                                         │
│     → route to Stage 3B (if extreme illumination)             │
│     → OR relax threshold to 0.1 and retry                    │
│     → OR flag as LOW_CONFIDENCE                               │
└─────────────────────────────────────────────────────────────┘
```

## Stage 3B: Extreme Illumination Path (CNSFM)

```
┌─────────────────────────────────────────────────────────────┐
│                  STAGE 3B INTERNAL FLOW                       │
│                                                               │
│  source_norm ──┐                                              │
│                ├──→ Crater Detector ──→ craters_src          │
│  reference_norm┘    (YOLOv9 fine-tuned    [(x,y,diameter)]  │
│                      on OHRC/TMC,                             │
│                      OR DeepMoon,                             │
│                      OR TMC-2 U-Net)                          │
│                      [CNSFM: github.com/Bin501/CNSFM]        │
│                                                               │
│  IF craters_src < 10 OR craters_ref < 10:                    │
│     → ABORT Stage 3B, fall back to Stage 3A                  │
│     (crater-sparse terrain)                                   │
│     [CNSFM §P4: fails in maria/melt ponds]                   │
│                                                               │
│  Build Crater Neighborhood Structure Features (CNSF):         │
│     For each crater: find K nearest neighbors                 │
│     Compute similarity-invariant parameters:                  │
│       angles between crater triplets                          │
│       side-length ratios                                      │
│       diameter ratios                                         │
│     [CNSFM: invariant under scale+rotation+translation]      │
│     [CNSFM: invariant under illumination BY CONSTRUCTION]    │
│                                                               │
│  Match CNSFs via NNDR on geometric distance (Eq.10)          │
│                                                               │
│  MCR (Mismatched CNSF Removal):                              │
│     Verify each match via local similarity transform          │
│     consistency with other matches                            │
│     [CNSFM §P1: RCM 62.2% → 100% in ablation]              │
│                                                               │
│  Output: sparse but high-reliability correspondences         │
│  (typically 10-50 matches, RCM ~100%)                        │
└─────────────────────────────────────────────────────────────┘
```

---

# SOFTWARE DEPENDENCY MAP

```
Core Dependencies (all open-source):
═══════════════════════════════════

Python 3.10+
├── numpy, scipy          ← FFT, linear algebra, paraboloid fitting
├── OpenCV (cv2)          ← image I/O, geometric transforms, RANSAC
├── GDAL / rasterio       ← geospatial cropping, CRS handling
├── pygeodesy             ← geodetic bounding-box computation
│
├── torch (PyTorch)       ← LightGlue + SuperPoint inference
│   └── hloc              ← SuperPoint + LightGlue wiring
│       (github.com/cvg/Hierarchical-Localization)
│
├── pydegensac            ← DEGENSAC robust estimation
│   (github.com/ducha-anh/pydegensac)
│
├── ANMS-Codes            ← Spatial distribution enforcement
│   (github.com/BAILOOL/ANMS-Codes)
│
├── skimage               ← phase_cross_correlation reference impl
│
└── [OPTIONAL]
    ├── CNSFM             ← Crater geometry matching
    │   (github.com/Bin501/CNSFM)
    ├── DeepMoon           ← Crater detection from DEMs
    │   (github.com/silburt/DeepMoon)
    └── ultralytics        ← YOLOv9 crater detection

External Tools (for data preparation):
═══════════════════════════════════════

├── USGS ISIS3            ← PDS4 ingestion, spiceinit, cam2map
│   (conda: usgs-astrogeology channel)
│
└── NASA ASP              ← Bundle adjustment, stereo, sub-pixel
    (stereopipeline.readthedocs.io)
    └── Used for: ground-truth generation, validation reference
```

---

# SINGLE-PAGE SPECIFICATION SUMMARY

```
╔════════════════════════════════════════════════════════════════╗
║  SIH 26166 — LUNAR IMAGE CORRESPONDENCE SYSTEM                ║
║  Architecture A (Recommended)                                  ║
╠════════════════════════════════════════════════════════════════╣
║                                                                ║
║  INPUT:  Chandrayaan-2 (OHRC/TMC/IIRS) + LRO (WAC/NAC)       ║
║                                                                ║
║  S1 NARROW:   pygeodesy bbox crop + difficulty classification  ║
║  S2 PREP:     ISIS3 cal → bilinear/bicubic resample            ║
║               → percentile-clip normalization → uint8          ║
║  S3 MATCH:    SuperPoint + LightGlue (primary)                ║
║               CNSFM crater geometry (extreme illum. fallback)  ║
║  S4 DISTRIBUTE: ANMS-SSC (uniform spatial coverage)           ║
║  S5 ESTIMATE:   DEGENSAC → polynomial fit → Z-score filter    ║
║  S6 REFINE:     Phase-correlation paraboloid sub-pixel         ║
║  S7 VALIDATE:   Confidence scoring → one-to-one → metrics     ║
║                                                                ║
║  OUTPUT: correspondences + transform + registered image        ║
║          + quality report (RMSE, inlier ratio, coverage, etc.) ║
║                                                                ║
║  TARGET: RMSE < 0.5 px (post-refinement, equatorial)          ║
║          Success rate > 95% equatorial, > 70% polar            ║
║          Uniform spatial coverage                              ║
║                                                                ║
║  KEY EVIDENCE:                                                 ║
║  • LightGlue: only method 6/6 success on our sensors          ║
║  • CNSFM: 72.3% SR at south pole (best: 31.2% alternatives)  ║
║  • Paraboloid: RMSE < 0.2 px on satellite imagery             ║
║  • DEGENSAC: guards against flat-terrain degeneracy            ║
║  • ANMS: O(n) confidence-weighted uniform distribution         ║
╚════════════════════════════════════════════════════════════════╝
```

---

# WHY THE AI SUGGESTION TO USE LNIFT WAS OVERRIDDEN

The [supplementary research](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/SIH26166_supplementary_research%282%29.md) recommended LNIFT as the primary correspondence method. This was a reasonable recommendation based on the evidence available at the time of that analysis (LNIFT: 100× faster than RIFT, 99.9% SR, built-in ANMS, spatial-domain illumination robustness).

However, the full research base — particularly the [Traditional_vs_DL](file:///c:/Workspace/code/Research/extracted/refiner/Traditional_vs_DeepLearning_FeatureMatching.md) paper which directly benchmarks on our exact Chandrayaan-2 sensors — provides stronger counter-evidence:

| Factor | LNIFT | LightGlue |
|--------|-------|-----------|
| Tested on OHRC-NAC (our sensor)? | ❌ No | ✅ Yes — RMSE 0.62/0.57 px |
| Tested on IIRS-WAC (our sensor)? | ❌ No | ✅ Yes — succeeded |
| Tested on polar lunar imagery? | ❌ No | ✅ Yes — only method to succeed |
| Tested on cross-modal (SAR-optical)? | ❌ No | ✅ Yes — DFSAR-SELENE |
| Evidence domain | Earth remote sensing | Our exact sensors + conditions |
| Speed | ✅ 0.49s (faster) | ✅ ~17ms on GPU (also fast) |

**Accuracy-first principle dictates:** when one method has direct evidence on our sensors/conditions and another does not, choose the one with evidence. LNIFT remains a strong comparison baseline and potential lightweight deployment alternative.

---

*This pipeline document consolidates the [main architecture](file:///C:/Users/poorn/.gemini/antigravity-ide/brain/dc39510c-c545-4417-acbd-91a8e64b3465/implementation_plan.md) and [companion document](file:///C:/Users/poorn/.gemini/antigravity-ide/brain/dc39510c-c545-4417-acbd-91a8e64b3465/architecture_companion.md) into a single operational reference.*
