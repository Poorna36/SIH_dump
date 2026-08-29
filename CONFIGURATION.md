# SIH26166 — CONFIGURATION v2.0

All tunable parameters. DO NOT change any value without reading its justification column.
Parameters marked (TUNE) require empirical validation on lunar data before finalizing.

---

## 1. Global Defaults (configs/default.yaml)

```yaml
global:
  seed: 42                  # fixed for all RANSAC and selection -- report in results
  coordinate_convention: xy  # col,row -- never row,col; see INTERFACES.md

data:
  raw_dir:        data/raw
  calibrated_dir: data/calibrated
  reference_dir:  data/reference
  pairs_dir:      data/pairs
  processed_dir:  data/processed
  results_dir:    results
  gt_dir:         data/metadata/gt
```

---

## 2. L0 — Data and Geometry (configs/ohrc_nac.yaml and tmc_wac.yaml)

### 2.1 ASP/ISIS
```yaml
asp:
  version_min: "3.7.0"                      # never lower -- Chandrayaan-2 camera fixes
  env: conda                                 # ASP bundled conda env
  isisdata: $HOME/projects/isisdata
  ck_window_days: 40                         # CK kernel window around strip UTC
  max_ck_size_gb: 5                          # per-window fetch limit (never 200 GB full set)

isisimport:
  preserve_filename: true                    # CRITICAL -- isisimport depends on original ISRO name

spiceinit:
  use_csm: auto                              # prefer CSM if model available, else ISIS kernels
```

### 2.2 Pair Building
```yaml
pair:
  k_pointing: 3                              # bbox padding multiplier; 2-5 range (TUNE per mission)
  sigma_pointing_m: 1000                     # typical OHRC pointing uncertainty ~500-2000m (TUNE)
  overlap_fraction_min: 0.5                  # below: allow partial, flag partial_overlap=true
  geo_cell_deg: 10                           # 10x10 degree cells for train/test split
  wac_reference: data/reference/wac_643nm.tif
  ode_timeout_s: 30                          # fallback to WAC if ODE times out
  strata:                                    # bins for reporting (never collapsed)
    lat_bins:
      equatorial:   [-45, 45]
      midlat:       [45, 60]
      polar:        [60, 90]               # note: >+/-55 lat triggers tile-wise models in L4
    az_bins:
      lt30:  [0, 30]
      lt60:  [30, 60]
      lt120: [60, 120]
      gt120: [120, 180]
    crater_density_bins:
      low:    [0, 2]
      medium: [2, 5]
      high:   [5, 100]
```

---

## 3. L1 — Preprocessing

```yaml
preprocessing:
  shadow_mask:
    incidence_threshold_deg: 80              # above = likely deep shadow; (TUNE for polar)
    local_variance_window: 15                # px window for flat-region detection
    flat_variance_threshold: 10              # below = featureless region
    mask_min_pct: 5                          # gate lower bound; if below, review detector
    mask_max_pct: 30                         # gate upper bound; if above, flag + proceed on unmasked

  radiometric_norm:
    percentile_clip: [2, 98]                # standard for cross-sensor normalization
    stat_transfer: true                      # mean/std of src matched to ref

  sensor_branches:
    ohrc_to_nac:
      clahe_clip_limit: 2.0                 # (TUNE for OHRC's 0.31m fine texture)
      clahe_tile_grid: [8, 8]
      pca_components: 1
      inversion: auto                        # flip if src/ref have inverted contrast
    tmc_to_wac:
      histogram_match: true
      clahe_clip_limit: 1.5                 # (TUNE)
      clahe_tile_grid: [8, 8]
      note: "experimental -- A/B test this branch vs. minimal preprocessing"
    learned_matchers:                        # M2 and M3
      mode: minimal                          # DO NOT apply heavy branches to learned matchers
      only_percentile_clip: true

  gsd:
    interpolation_low_angle: bilinear        # low solar elevation (high shadow): bilinear
    interpolation_high_angle: bicubic        # high solar elevation (crisp): bicubic
    low_angle_threshold_deg: 45              # incidence > 45deg = low angle (TUNE)

  tiling:
    size_px: 512                             # standard; reduce to 256 for GPU memory limits
    overlap_px: 64                           # ensures feature continuity at borders
    min_tile_fraction: 0.5                   # discard tiles smaller than half the grid
```

---

## 4. L2 — Correspondence Engine

```yaml
matchers:
  # M0: SIFT
  sift:
    nfeatures: 0                             # 0 = unlimited before ANMS
    noctave_layers: 3
    contrast_threshold: 0.04
    edge_threshold: 10
    sigma: 1.6
    lowe_ratio: 0.75                         # research standard (SIFT-IIRS-WAC)
    anms:
      enabled: true
      budget: 2000                           # pre-match budget per tile (TUNE)
      method: ssc                            # Suppression via Square Covering (fastest)
    ransac:
      method: degensac
      threshold_px: 3.0                      # starting threshold; ladder adjusts
      max_iter: 10000
      confidence: 0.99999

  # M1: RIFT2 + scale extension
  rift2:
    num_scales: 4                            # Ns in log-Gabor bank
    num_orientations: 6                      # No in log-Gabor bank; multi-MIM cost = No x descriptor
    scale_space_octaves: 4                   # our extension -- multi-octave for scale invariance
    pc_threshold: 0.1                        # phase congruency keypoint threshold (TUNE)
    mim_size: 96                             # descriptor spatial support J
    anms:
      enabled: true
      budget: 1500
      method: ssc
    note: "15-30x slower than SURF -- benchmark per-pair; one documented total failure"

  # M2: SuperPoint + LightGlue
  lightglue:
    backbone: superpoint                     # superpoint or disk
    max_keypoints: 2048                      # kp_limit; reduce to 1024 on CPU
    match_threshold: 0.0                     # let LightGlue confidence filter handle this
    depth_confidence: 0.95                   # early stopping
    width_confidence: 0.99                   # early stopping
    requires_gpu: true                       # preferred; cpu_fallback=true falls to CPU
    cpu_fallback: true
    f2_checks: mandatory                     # in-domain bounds + one-to-one -- NEVER disable

  # M3: Crater-geometry (CNSFM-style)
  crater:
    detector: yolov9                         # pretrained, fine-tuned via transfer learning
    min_crater_diameter_px: 8               # below this, skip crater as too small
    crater_density_gate: 3.0               # tau_c: both images must exceed this value (TUNE)
    mcr_outlier_method: mcr_structural
    topology_similarity_threshold: 0.65     # (TUNE)
    gate_terrain_classes: [highland, polar_highland, polar]
    note: "explicitly fails in crater-sparse mare -- gating is mandatory, not optional"

arbitration:
  always_run: [sift]                         # M0 always runs as floor + fallback
  inlier_ratio_floor: 0.05                   # below = primary matcher failed, use fallback
  fallback_to: sift                          # M0 is the fallback
  preference_order: [crater, lightglue, rift2, sift]
```

---

## 5. L3 — Uniform Correspondence Optimization

```yaml
selection:
  confidence_filter:
    sift: 0.0                                # lowe ratio already applied; keep all
    rift2: 0.0                               # NCC-based; TUNE if many outliers
    lightglue: 0.2                           # LightGlue confidence score threshold (TUNE)
    crater: 0.65                             # topology similarity score

  grid:
    n_rows: 8                                # NxN = 8x8 grid over source image
    n_cols: 8
    cap_per_cell: 5                          # max matches per cell; relax once on gate failure

  budget: 250                                # target match count after selection (TUNE)
  min_matches_after_selection: 25            # gate; if below: relax cap once, then fail
  coverage_min: 0.60                         # gate; fraction of valid cells that have >= 1 match

  conflict_resolution: highest_confidence    # on one-to-many merge conflicts

  report:
    coverage_before: true                    # mandatory
    coverage_after: true                     # mandatory
    grid_density_std: true                   # mandatory metric
```

---

## 6. L4 — Geometric Verification

```yaml
verification:
  f2_checks:
    enabled: mandatory                       # always enforced, especially for M2/M3
    in_domain_buffer_px: 10                  # match must land inside image + buffer
    one_to_one: strict                       # no duplicate src or ref points

  ransac:
    method: degensac                         # degeneracy-aware; MAGSAC++ also acceptable
    max_iter: 10000
    confidence: 0.99999
    threshold_px: auto                       # see t_gsd formula below
    widen_on_failure: 1.5                    # widen threshold once before tile-wise fallback

  # t_gsd formula: max(0.5, gsd_ratio * 1.0) up to 3.0 px
  t_gsd:
    min_px: 0.5
    max_px: 3.0
    gsd_multiplier: 1.0

  model_ladder:
    - similarity                             # 4 dof
    - affine                                 # 6 dof
    - homography                             # 8 dof
    stop_on_rmse_below: 1.0                  # accept simplest passing model (TUNE)

  tilewise:
    enabled: true
    trigger_latitude_deg: 55                 # >+/-55 latitude OR high-relief
    trigger_relief_m: 500                    # (TUNE -- requires DEM probe)
    tile_size_px: 512
    overlap_px: 256                          # 50% overlap
    min_inliers_per_tile: 12

  gcp_declustering:
    min_spacing_px: 20
    method: grid_nearest_center
    zscore_threshold: 3.0
    min_gcps_for_zscore: 20

  desca:
    enabled: false                           # optional; default off
    nndr_strict: 0.7                         # seed set
    nndr_loose: 1.0                          # evaluation pool
    de_generations: 200
    de_F: 0.9
    de_Cr: 0.9
    note: "random DE init is a demonstrated failure -- two-tier init is mandatory"
    geometry_restriction: affine_only        # skip if strong relief or homography required
```

---

## 7. L5 — Sub-pixel Refinement

```yaml
refinement:
  method: ncc                                # ncc or poc (phase-only correlation)
  window_px: 32                              # search window W around coarse match
  apodization: tukey                         # tukey or gaussian; NEVER blackman
  tukey_alpha: 0.5
  pyramid_levels: 3                          # Gaussian pyramid coarse-to-fine
  peak_fit: paraboloid                       # 2D paraboloid sub-pixel peak fitting
  sharpness_threshold: 0.15                  # P[0,0]/sum(3x3) < tau_q -> reject; (TUNE)
  min_refined_fraction: 0.70                 # gate; below = flag pair as partial_refinement
  report_before_after_rmse: true             # mandatory
```

---

## 8. L6 — Product Generation

```yaml
products:
  registered_format: GeoTIFF
  resampling: bicubic
  valid_warp_fraction_min: 0.90              # gate; below = report partial extent
  gcp_format: csv                            # + GDAL .gcp sidecar
  qc_artifacts:
    - checkerboard_overlay
    - match_overlay
    - residual_heatmap
  qc_checkerboard_tile_px: 64
  residual_heatmap_sigma: 3                  # px; heat is per-match residual
```

---

## 9. L7 — Evaluation

```yaml
evaluation:
  gt:
    grid_size: 6                             # 6x6 = 36 checkpoints per pair
    pairs_to_annotate: [15, 20]             # min and target count
    min_points_per_pair: 30
    qc_reannotate_pct: 0.20                  # 20% of points re-annotated for error bound
    eval_partition: eval                     # only "eval" partition points used for RMSE

  leakage:
    split_by: geo_cell                       # disjoint 10x10 degree cells
    geo_cell_size_deg: 10
    audit_required: true                     # leakage audit must pass before publishing

  metrics:
    primary:
      - rmse_px
      - pct_lt_1px
      - pct_lt_0p5px
      - inlier_ratio
    secondary:
      - medae_px
      - inlier_count
      - spatial_coverage
      - grid_density_std
      - refinement_gain_px
      - runtime_s
    optional:
      - precision
      - recall
      - matching_score

  reporting:
    stratify_by: [sensor_pair, terrain_class, latitude_bin, delta_az_bin]
    never_hide: [polar, high_latitude, partial_overlap]
    aggregate: [mean, median]
```

---

## 10. IIRS Parallel Track (configs/iirs_wac.yaml)

```yaml
extends: default

iirs:
  format: qub
  photometric_correction: true              # mandatory before any feature operation
  correction_model: hapke                   # (TUNE -- verify against IIRS calibration report)
  registration_band: auto                   # auto = brightest band near WAC 643nm
  reference: wac
  accuracy_target_m: 80                    # sub-pixel at 80m GSD = sub-80m absolute RMSE
  separate_module: true                     # NEVER fold into main ohrc_nac / tmc_wac pipeline
```

---

## Notes on (TUNE) Parameters

The following parameters were set from literature and need empirical validation on actual lunar data:
- shadow_mask.incidence_threshold_deg (polar night boundary is very sharp)
- shadow_mask.flat_variance_threshold (lunar maria are very flat)
- anms.budget (depends on actual scene texture density)
- lightglue.match_threshold (domain gap may shift optimal value)
- crater.crater_density_gate tau_c (count distribution unknown for our tile sizes)
- crater.topology_similarity_threshold (test on manually verified crater pairs)
- model_ladder.stop_on_rmse_below (set to 1.0 px as starting point; may relax for IIRS pairs)
- tilewise.trigger_relief_m (needs a DEM probe -- approximate from known crater depths initially)
- refinement.sharpness_threshold (depends on actual patch contrast distribution in our data)

Benchmark these explicitly on the 3-pair pilot before running the full dataset.
