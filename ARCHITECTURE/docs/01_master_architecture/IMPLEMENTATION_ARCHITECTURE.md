# LUNAR MULTI-MODAL IMAGE CORRESPONDENCE ARCHITECTURE
## SIH Problem Statement 26166

---

# 1. EXECUTIVE ARCHITECTURE

A **coarse-to-fine, hybrid classical+learned pipeline** that uses metadata-driven spatial narrowing, illumination-invariant feature detection via Phase Congruency, context-aware learned matching via LightGlue, spatially-enforced correspondence selection via Adaptive NMS, robust geometric estimation via DEGENSAC, and dedicated sub-pixel refinement via phase-correlation paraboloid fitting — with a conditional crater-geometry branch (CNSFM) activated for extreme illumination cases where appearance-based methods degrade.

The system treats the problem as a **six-stage cascade**: NARROW → DETECT → MATCH → FILTER → REFINE → VALIDATE — where each stage addresses a specific failure mode that the previous stage cannot handle alone.

---

# 2. WHY THIS ARCHITECTURE

## Central reasoning

The research base reveals a clear hierarchy of difficulty in lunar image correspondence:

1. **Equatorial, same-sensor, moderate Sun-angle** — classical SIFT works adequately (RMSE ~1-6 px) [SIFT-IIRS-WAC, MoonMetaSync, Traditional_vs_DL]
2. **Cross-sensor, moderate illumination** — RIFT2/phase-congruency methods handle radiometric differences but struggle with geometric distortion [RIFT, KAZE, Traditional_vs_DL]
3. **Polar, extreme Sun-angle, cross-modal** — only SuperGlue/LightGlue succeeded across ALL tested conditions including polar OHRC-NAC and DFSAR-SELENE [Traditional_vs_DL]
4. **Extreme azimuth difference (near-180°), shadow-dominated** — even SuperGlue reaches limits; only geometric/topological methods (CNSFM) maintain 72.3% success where all appearance-based methods fail [CNSFM]

This hierarchy demands a **layered approach**: a primary learned matcher (LightGlue) that handles conditions 1–3, with a geometric-landmark fallback (CNSFM-style) for condition 4, rather than trying to build a single monolithic method.

The architecture is NOT "SuperGlue + everything else." It is structured around **complementary failure modes**:

- **LightGlue** fails when the front-end detector (SuperPoint) cannot find repeatable keypoints — in low-texture mare or extreme shadow [SuperGlue §P4].
- **Phase Congruency (PC) detection** is illumination-invariant by construction and detects where intensity-based detectors fail [RIFT], but its descriptor (MIM) lacks the contextual reasoning that gives learned matchers their accuracy advantage.
- **CNSFM** bypasses pixel appearance entirely using crater geometry, succeeding under extreme illumination — but fails in crater-sparse terrain [CNSFM §P4].
- **Phase-correlation sub-pixel refinement** provides the final sub-pixel accuracy that neither feature matching nor learned matchers reliably achieve alone [HybridPhaseCorrelation].

Each component addresses a gap the others leave open.

## Why not a single method?

No single method in the research base satisfies ALL requirements simultaneously:

| Method | Illumination | Scale | Multi-modal | Low texture | Sub-pixel | Spatial dist. |
|--------|-------------|-------|-------------|-------------|-----------|---------------|
| SIFT | ❌ | ✅ | ❌ | ❌ | ❌ | ❌ |
| RIFT/RIFT2 | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| SuperGlue/LightGlue | ✅ | ✅ | ✅ | ⚠️ | ⚠️ | ⚠️ |
| CNSFM | ✅✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Phase correlation | ⚠️ | ❌ | ⚠️ | ✅ | ✅✅ | N/A |

The architecture combines them so their ✅ cells cover each other's ❌ cells.

---

# 3. FULL PIPELINE

```
┌─────────────────────────────────────────────────────────────────┐
│                        INPUT STAGE                               │
│  Chandrayaan-2 (OHRC / TMC / IIRS) + Reference (LRO NAC/WAC)   │
│  + Metadata (corner coords, solar angles, sensor params)         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1: METADATA-DRIVEN SPATIAL NARROWING                      │
│  Geodetic footprint query → padded bounding box crop              │
│  Solar-angle metadata extraction → difficulty classification      │
│  Resolution ratio computation → resampling strategy selection     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 2: PREPROCESSING & NORMALIZATION                          │
│  ISIS3 calibration (if raw PDS4)                                  │
│  Resolution harmonization (bilinear/bicubic adaptive)            │
│  Percentile-clip intensity normalization (2nd/98th → [0,1])      │
│  8-bit conversion                                                 │
│  [NO heavy enhancement — evidence shows it doesn't help]         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                  ┌────────┴────────┐
                  │  ROUTE DECISION  │
                  │  (Solar angle    │
                  │   difference     │
                  │   assessment)    │
                  └───┬─────────┬───┘
          ≤ 60° diff  │         │  > 60° diff OR
          (PRIMARY)   │         │  polar region
                      ▼         ▼  (EXTREME)
┌──────────────────────┐  ┌──────────────────────┐
│ STAGE 3A: PRIMARY    │  │ STAGE 3B: EXTREME    │
│ CORRESPONDENCE       │  │ ILLUMINATION PATH    │
│                      │  │                      │
│ SuperPoint detection │  │ Crater detection     │
│ + LightGlue matching │  │ (YOLOv9/DeepMoon)   │
│                      │  │ + CNSFM geometric    │
│ Confidence threshold │  │   topology matching  │
│ (dustbin > 0.2)      │  │ + MCR outlier        │
│                      │  │   removal            │
│ ALSO: PC-weighted    │  │                      │
│ keypoints as         │  │ IF insufficient      │
│ supplementary        │  │ craters → fallback   │
│ candidates           │  │ to Stage 3A          │
└──────────┬───────────┘  └──────────┬───────────┘
           │                         │
           └────────┬────────────────┘
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 4: SPATIAL DISTRIBUTION ENFORCEMENT                       │
│  Adaptive Non-Maximal Suppression (ANMS/SSC)                     │
│  Target: uniform coverage with confidence-weighted retention      │
│  Fallback: grid-based top-k per cell                              │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 5: ROBUST GEOMETRIC ESTIMATION                            │
│  DEGENSAC (degeneracy-aware RANSAC)                               │
│  Similarity/affine transform estimation per tile                  │
│  Global polynomial fit on merged inliers                          │
│  Residual Z-score outlier filter (> 2.5σ removed)                │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 6: SUB-PIXEL REFINEMENT                                   │
│  Per-correspondence local phase correlation                       │
│  (Gaussian-windowed, 3×3 paraboloid peak fitting)                 │
│  Reject if refinement shift > 2 px or correlation < threshold    │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 7: CONFIDENCE ESTIMATION & FINAL VALIDATION               │
│  Per-match confidence = f(matcher_score, RANSAC_residual,        │
│                           refinement_correlation, spatial_info)   │
│  Final one-to-one enforcement                                     │
│  Output: correspondence set + transform + quality metrics         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                         OUTPUT                                    │
│  • Filtered correspondence point set (source ↔ reference)        │
│  • Geometric transformation parameters                            │
│  • Registered/warped source image in reference CRS               │
│  • Per-match confidence scores                                    │
│  • Quality report: RMSE, inlier count/ratio, spatial coverage,   │
│    sub-pixel accuracy %, MedAE                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

# 4. COMPONENT-BY-COMPONENT DESIGN

---

## STAGE 1: Metadata-Driven Spatial Narrowing

**Input:** Raw source image (OHRC/TMC/IIRS) + reference dataset (LRO WAC global mosaic / LRO NAC strips) + spacecraft metadata (corner lat/lon, solar angles).

**Method:** Geodetic bounding-box query using corner coordinates from metadata → crop reference image with 2–5× padding beyond pointing uncertainty.

**Output:** Cropped, co-approximate source–reference image pair with known approximate overlap; solar-angle difference and resolution ratio metadata.

**Why this method:** [MoonMetaSync] demonstrated OHRC↔TMC-2 geodesic pre-matching using `pygeodesy` + Haversine distance on the exact sensor pair we need. [SIH26166_reference_mapping] confirms the pattern: "metadata narrows planet-wide uncertainty to a padded patch → feature matching narrows to sub-pixel." Without this, we would search the entire reference mosaic — computationally infeasible and producing massive false-match rates.

**Alternatives rejected:**
- *Metadata-only georeferencing* [SIFT-IIRS-WAC §P4]: produces systematic offsets; metadata alone is not accurate enough.
- *Full-image matching without narrowing*: computationally prohibitive for OHRC at 0.25 m resolution.

**Failure mode:** Metadata corner coordinates are inaccurate (known issue, [RIFT §P4: "satellite geolocation can be off by hundreds of meters"]). Mitigation: generous padding (5× uncertainty) ensures overlap.

**Computational cost:** Negligible — bounding-box crop of a pre-downloaded reference mosaic.

**Feasibility:** HIGH. Uses standard geospatial libraries (pygeodesy, GDAL/rasterio). Metadata fields exist in OHRC/TMC/IIRS PDS products.

---

## STAGE 2: Preprocessing & Normalization

**Input:** Cropped source and reference image patches.

**Method:**
1. ISIS3 calibration (if ingesting raw PDS4): radiometric correction + SPICE-based geometry (`spiceinit`).
2. Resolution harmonization: resample the lower-resolution image to match the higher-resolution one. Use **bilinear interpolation** for low Sun-angle images, **bicubic** for high Sun-angle images (evidence-based choice from [MoonMetaSync §P2]).
3. Percentile-clip intensity normalization: 2nd/98th percentile clip → scale to [0,1] → statistic transfer (mean/std matching of source to reference) → uint8 conversion. Directly from [SIFT-IIRS-WAC §P0].
4. **No heavy enhancement pipeline.** [Traditional_vs_DL §P2] demonstrated that heavy preprocessing (CLAHE + inversion + dilation + PCA) does not rescue classical methods in polar conditions. [Traditional_vs_DL §P2] further shows "algorithms that do not necessitate explicit preprocessing can indeed outperform traditional methods that do."

**Output:** Normalized, resolution-matched image pair ready for feature detection.

**Why this method:** The percentile-clip + statistic-transfer approach is directly validated on IIRS↔WAC [SIFT-IIRS-WAC] and implicitly supports OHRC↔NAC pairs. Heavy enhancement is explicitly rejected based on evidence: it adds complexity without improving polar/extreme performance.

**Alternatives rejected:**
- *CLAHE + shadow normalization + log transform* [Traditional_vs_DL]: demonstrated insufficient for polar conditions despite significant engineering effort.
- *cGAN radiometric normalization* [Radiometric_Normalization]: requires paired training data that doesn't exist for our sensor combinations; solves mosaicking, not registration; assumes pre-registered images.
- *Histogram matching / Retinex* [Radiometric_Normalization §P4]: "limited in handling nonlinear variations caused by changes in illumination geometry."

**Failure mode:** Very large resolution ratios (OHRC 0.25m vs IIRS 80m = 320×) may cause resampling to destroy discriminative texture. **Mitigation:** For extreme ratios (>10×), treat as a separate module (see §IIRS handling below).

**Computational cost:** Low. Standard image processing operations.

**Feasibility:** HIGH.

---

## STAGE 3A: Primary Correspondence — LightGlue

**Input:** Normalized, resolution-matched image pair.

**Method:** SuperPoint keypoint detection → LightGlue matching (successor to SuperGlue).

**Output:** Set of candidate correspondences with per-match confidence scores.

**Why LightGlue (not SuperGlue, not RIFT, not SIFT):**

| Criterion | LightGlue/SuperGlue | RIFT2 | SIFT |
|-----------|---------------------|-------|------|
| Polar success | ✅ Only method succeeding on ALL 6 conditions [Traditional_vs_DL] | ❌ Failed OHRC-NAC Polar [Traditional_vs_DL] | ❌ Failed polar [Traditional_vs_DL] |
| Cross-modal | ✅ Succeeded DFSAR-SELENE [Traditional_vs_DL] | ❌ Failed DFSAR-SELENE Eq [Traditional_vs_DL] | ❌ Failed SAR [Traditional_vs_DL] |
| RMSE accuracy | ✅ Best: 0.4–0.9 px [Traditional_vs_DL] | ⚠️ 1.5/1.2 px [Traditional_vs_DL] | ❌ 3.6/5.9 px [Traditional_vs_DL] |
| Runtime | ✅ Single-digit seconds [Traditional_vs_DL] | ⚠️ 36.9s [Traditional_vs_DL] | ❌ 678–809s [Traditional_vs_DL] |
| Illumination (day-night) | ✅ Demonstrated generalization [SuperGlue §P0] | ✅ By design [RIFT] | ❌ "Most sensitive to illumination" [CNSFM] |
| Training data dependency | ⚠️ Needs domain-specific fine-tuning | ✅ None | ✅ None |

**Why LightGlue over SuperGlue specifically:** LightGlue [Additional_Resources §11] is SuperGlue's official successor — same core attention+optimal-transport architecture but with adaptive depth/width (early-stopping), ~16.9 ms for easy pairs, supports SuperPoint/DISK/ALIKED/SIFT front ends, Apache-2.0 licensed, integrated into `hloc` toolbox [Additional_Resources §10].

**Key architectural decisions within this stage:**

1. **SuperPoint as front-end detector:** Best co-designed pairing with LightGlue [SuperGlue §P1]. The dustbin mechanism [SuperGlue §P0] explicitly handles unmatched keypoints, providing a first layer of outlier rejection.

2. **Confidence threshold at 0.2:** Below this, matches are assigned to the "dustbin" (no match). This is not arbitrary — it's the paper-validated threshold [SuperGlue §P0].

3. **Supplementary PC-weighted keypoints:** In addition to SuperPoint, extract Phase Congruency keypoints [RIFT §P0] and feed them as supplementary candidates to LightGlue. **Rationale:** SuperPoint is trained on natural images and may miss illumination-invariant keypoints on lunar terrain. PC detection is illumination-invariant by construction [RIFT §P2: "measures local phase alignment across scales/orientations, not intensity"]. LightGlue is front-end agnostic [SuperGlue §P1] and can accept mixed keypoint sources.

   > [!IMPORTANT]
   > This is a **[PROPOSED INTEGRATION]** — using PC-detected keypoints as input to LightGlue has not been demonstrated in any reviewed paper. It is architecturally plausible because LightGlue accepts keypoints + descriptors from any source, and PC+MIM descriptors have compatible dimensionality. However, it requires validation.

**Failure mode:** SuperPoint fails to detect repeatable keypoints in low-texture lunar mare or extreme shadow. [SuperGlue §P2: "failure cases occur either due to unlikely motion or lack of repeatable keypoints"]. **Mitigation:** The supplementary PC keypoints provide a second detection path; if both fail, the system falls through to Stage 3B (crater geometry).

**Computational cost:** Moderate. LightGlue: ~17 ms per easy pair, up to ~100 ms for hard pairs (GPU required). SuperPoint: ~50 ms. PC extraction: ~200 ms. Total: <500 ms per pair on GPU.

**Feasibility:** HIGH. Pretrained SuperPoint + LightGlue weights are publicly available. Fine-tuning on lunar data is recommended but not strictly required for initial deployment.

---

## STAGE 3B: Extreme Illumination Path — CNSFM

**Input:** Normalized image pair flagged as "extreme" (solar azimuth difference > 60° or polar region).

**Method:** CNSFM (Crater Neighborhood Structure Feature Matching) — YOLOv9 crater detection → K-nearest-neighbor crater topology → similarity-invariant geometric matching → MCR outlier removal.

**Output:** Sparse but highly reliable correspondence set based on crater geometry.

**Why this component exists:** [CNSFM] demonstrated 72.3% success rate at the lunar south pole under up to 147.7° azimuth difference — where the best alternative achieved only 31.2%. The method is illumination-invariant **by construction** (not by engineering) because it matches geometric topology (crater center positions + diameters), not pixel intensities.

**Why it is NOT the primary path:** 
1. **Crater-sparse terrain failure:** [CNSFM §P4] explicitly acknowledges failure in "geologically young units (e.g., light plains or impact melt ponds)" — crater geometry is not universal.
2. **Low match count:** CNSFM produces tens of matches (not hundreds) due to reliance on crater density [CNSFM §P3].
3. **Higher RMSE:** Average 1.0–2.2 px versus SuperGlue's ~0.4–0.9 px [CNSFM §P0 vs Traditional_vs_DL §P0].

**When activated:** Conditionally, based on metadata solar-angle assessment or when Stage 3A produces fewer than a minimum viable number of correspondences (e.g., <15 matches).

**Fallback:** If insufficient craters are detected (<10 per image), falls back to Stage 3A with relaxed confidence thresholds.

**Implementation:** The CNSFM code + dataset is publicly available at `github.com/Bin501/CNSFM`. The crater detector needs fine-tuning on OHRC/TMC imagery (currently trained on LROC NAC at 0.5m). [Additional_Resources §12-13] identifies two ready-made crater detectors: DeepMoon (LOLA/WAC DEM trained) and a TMC-2-specific crater U-Net.

**Failure mode:** Crater-sparse terrain (mare, melt ponds) — no craters to match. Near-total shadow coverage of overlap region [CNSFM §P2].

**Computational cost:** YOLOv9 inference: ~100 ms. CNSFM matching: <1s. Total: <2s.

**Feasibility:** MEDIUM-HIGH. Public code exists. Crater detector retraining on OHRC/TMC is the main effort.

---

## STAGE 4: Spatial Distribution Enforcement

**Input:** Raw correspondence set from Stage 3A or 3B.

**Method:** Adaptive Non-Maximal Suppression (ANMS) using the SSC (Suppression via Square Covering) variant.

**Output:** Spatially uniform subset of correspondences, retaining the locally strongest match per spatial cell.

**Why this is a dedicated stage:** The problem statement **explicitly** requires "a UNIFORM SPATIAL DISTRIBUTION of reliable match points." No paper in our research base addresses this as part of the matching algorithm itself. [Supplementary_Research §1] identifies ANMS as the standard solution: "instead of keeping the top-N strongest keypoints (which clusters around a few high-contrast craters), each keypoint is suppressed if a stronger keypoint exists within some radius."

**Why ANMS over simpler grid-based selection:**
- ANMS is confidence-weighted — retains the strongest match per neighborhood rather than an arbitrary one per grid cell.
- SSC variant is O(n) and readily available [Supplementary_Research §1]: `github.com/BAILOOL/ANMS-Codes`.
- [LoFTR_IMC21 §P1] describes a similar adaptive-NMS via bisection-searched suppression radius — confirms the pattern's value in the transformer-matcher context.

**Fallback:** If ANMS implementation proves problematic, use grid-based top-k per cell (divide image into N×N grid, keep top-k matches by confidence per cell). [SIH26166_supplementary §1] notes this is "cruder but trivial to implement."

**What we removed:** The supplementary research recommended LNIFT (which has ANMS built in) as the primary method. We rejected this because LNIFT's match accuracy evidence is less comprehensive than LightGlue's across our actual failure conditions (polar, cross-modal), despite its speed advantage. However, LNIFT's built-in ANMS principle is adopted as a standalone post-processing step.

**Failure mode:** Over-suppression leaving too few matches for geometric estimation. **Mitigation:** Set minimum match-count floor; if ANMS reduces below this, widen suppression radius.

**Computational cost:** <10 ms. Negligible.

**Feasibility:** HIGH. Ready-to-use implementations exist.

---

## STAGE 5: Robust Geometric Estimation

**Input:** Spatially distributed correspondence set from Stage 4.

**Method:** Three-layer estimation:
1. **DEGENSAC** (degeneracy-aware RANSAC): estimate similarity/affine transform with 0.5 px reprojection threshold, up to 10,000 iterations, 0.99999 confidence.
2. **Tile-merge + global polynomial fit**: after per-tile DEGENSAC, merge inliers and fit a global polynomial transformation by least squares.
3. **Residual Z-score filter**: remove matches with residual > 2.5σ from the fitted model (requires >20 GCPs). Directly from [SIFT-IIRS-WAC §P0].

**Output:** Geometric transformation parameters + filtered inlier correspondence set.

**Why DEGENSAC (not standard RANSAC, not DESCA):**
- **vs. standard RANSAC:** [LoFTR_IMC21 §P0] and [HybridPhaseCorrelation §P0] both use RANSAC but without degeneracy handling. Lunar terrain includes large flat regions (mare, crater floors) where a dominant-plane degeneracy can cause standard RANSAC to estimate a degenerate model. DEGENSAC explicitly guards against this [LoFTR_IMC21 §P0].
- **vs. DESCA (DE-based optimization):** DESCA [DESCA §P0] achieves better RMSE (0.67–0.95 px) than RANSAC-family methods, but (a) only demonstrated on optical-optical same-modality pairs, (b) only with affine model which [DESCA §P4] "fails on mountainous terrain," and (c) requires careful two-tier initialization that adds complexity. DESCA is classified as **HIGH-VALUE OPTIONAL** — worth investigating in Phase 3 as a DEGENSAC replacement if accuracy demands it.

**Why polynomial (not pure homography):**
- [SIFT-IIRS-WAC §P1] decouples verification (per-tile homography RANSAC) from final warp (polynomial), and this is validated on real Chandrayaan-2 data.
- [DESCA §P4] demonstrates "near-total failure on mountainous terrain" with a global affine model — lunar craters and mountains have substantial local relief.
- [SIFT-IIRS-WAC §P2] shows error grows near-exponentially beyond ±55° latitude with polynomial/homography models — for high-latitude data, a piecewise or higher-order model may be needed (flagged as future work).

**Failure mode:** Insufficient inliers (<8) for robust model fitting. **Mitigation:** Relax DEGENSAC threshold from 0.5 to 1.0 px; if still insufficient, fall back to similarity (4-param) rather than affine (6-param) model.

**Computational cost:** DEGENSAC: ~100–500 ms depending on point count. Polynomial fit: <10 ms.

**Feasibility:** HIGH. DEGENSAC is available in OpenCV (`cv2.findFundamentalMat` with USAC_DEGENSAC flag) and pydegensac library.

---

## STAGE 6: Sub-Pixel Refinement

**Input:** Inlier correspondences from Stage 5, plus the original image pair.

**Method:** Per-correspondence local phase correlation with sub-pixel paraboloid fitting:
1. For each inlier match, extract a local patch (32×32 or 64×64 px) centered on the match location in both images.
2. Apply Gaussian windowing (not Blackman — [HybridPhaseCorrelation §P4]: "Blackman performs worst because it suppresses too much high-frequency information").
3. Compute phase-only cross-correlation in the frequency domain.
4. Fit a 2D paraboloid to the 3×3 neighborhood of the correlation peak.
5. The paraboloid vertex gives the sub-pixel displacement correction.
6. **Reject** if: refinement shift > 2 px (inconsistent with the coarse match) OR peak correlation < 0.3 (unreliable).

**Output:** Sub-pixel-refined correspondence set.

**Why phase correlation + paraboloid fitting:**
- [HybridPhaseCorrelation §P0] demonstrated RMSE below 0.2 px (Sentinel-2) and 0.4 px (Pleiades) using exactly this approach.
- [HybridPhaseCorrelation §P2] tested seven sub-pixel strategies; paraboloid fitting achieved the best result (RMSE 0.010 px, 96.3% improvement over integer matching).
- Phase correlation operates in the frequency domain and is inherently robust to uniform intensity differences — complementary to the spatial-domain feature matching.
- This is computationally cheap (FFT of small patches) and adds no training data dependency.

**Why NOT other refinement methods:**
- *MI + DE optimization* [KAZE §P0]: RMSE 0.57–0.82 px — competitive but adds significant compute (200 DE generations per match) and is only validated with affine model.
- *LoFTR's learned coarse-to-fine refinement*: achieves sub-pixel via neural upsampling, but [HybridPhaseCorrelation §P4] reports LoFTR "produces out-of-image-domain correspondences" — less reliable than analytical fitting.
- *OpenCV cornerSubPix*: simplistic for our use case; designed for corner refinement, not general correspondence refinement.

**Failure mode:** Phase correlation fails on patches with insufficient texture (flat mare) or strong aliasing. **Mitigation:** Reject refinements with low peak correlation; retain the coarse match location for these points (still valid, just not sub-pixel refined).

**Computational cost:** ~1 ms per correspondence (FFT of 64×64 patch × 2 + peak fitting). For 200 matches: ~200 ms total.

**Feasibility:** HIGH. Implementable using numpy/scipy FFT. Reference implementations exist in `skimage.registration.phase_cross_correlation`.

---

## STAGE 7: Confidence Estimation & Final Validation

**Input:** Sub-pixel-refined correspondences with associated metadata.

**Method:**
1. Per-match composite confidence score combining:
   - LightGlue matching score (or CNSFM geometric distance)
   - DEGENSAC reprojection residual
   - Phase-correlation peak value
   - Local texture measure (variance of the source patch)
2. One-to-one enforcement: if multiple source points map to the same reference location, retain only the highest-confidence match.
3. Final quality metrics computation.

**Output:**
- Final correspondence point set: `[(x_src, y_src, x_ref, y_ref, confidence)]`
- Geometric transformation parameters (polynomial coefficients)
- Registered/warped source image in reference CRS
- Quality report: RMSE, inlier count, inlier ratio, spatial coverage metric, sub-pixel accuracy %, MedAE

**Why this stage exists:** The problem statement requires "measurable outputs" and "appropriate metrics such as RMSE, inlier count, inlier ratio." This stage transforms raw correspondences into the deliverable.

**Computational cost:** Negligible.

**Feasibility:** HIGH.

---

# 5. HOW EACH REQUIREMENT IS SOLVED

| Requirement | Architectural Component | How It Is Handled | Remaining Limitation |
|---|---|---|---|
| **Illumination / Sun-angle** | LightGlue (learned illumination generalization [SuperGlue §P0]) + CNSFM fallback (geometry-only, illumination-invariant by construction [CNSFM §P0]) | Primary: attention-based learned matching generalizes across day↔night [SuperGlue Aachen]. Extreme: crater geometry bypasses pixel appearance entirely | Extreme shadow overlap loss (>90% shadow) defeats both [CNSFM §P2] |
| **Scale variation** | Stage 2 resolution harmonization + LightGlue multi-scale matching | Resolution resampled before matching. LightGlue handles residual scale via learned features | Extreme ratios (>10×, e.g. OHRC↔IIRS) may need dedicated coarse alignment |
| **Rotation** | LightGlue handles arbitrary rotation [SuperGlue Table 1]. RIFT's multi-MIM provides backup rotation invariance [RIFT §P0] | Learned matching inherently rotation-robust via training. CNSFM is similarity-invariant | None significant for in-plane rotation |
| **Viewpoint / geometric** | LightGlue cross-attention [SuperGlue §P2]. DEGENSAC handles projective geometry [LoFTR_IMC21 §P0] | Attention-based context captures geometric relationships. DEGENSAC filters geometrically inconsistent matches | Very large viewpoint changes (>60° convergence angle) may require explicit stereo modeling |
| **Multi-modality** | LightGlue (demonstrated on DFSAR-SELENE [Traditional_vs_DL]). PC keypoints as supplement (illumination-invariant by construction [RIFT]) | LightGlue is the only tested method succeeding across all modalities. PC supplements where learned features are weak | IIRS hyperspectral (80m, 250 bands) is a fundamentally different sub-problem — treat as separate module |
| **Low texture** | LightGlue's global context via cross-attention [SuperGlue §P2]. Phase correlation refinement works on low-texture patches via frequency domain | Attention aggregates context from the entire image, not just local patches. Phase correlation detects correlation peaks even in subtle texture | If both detection AND correlation fail (perfectly flat mare), no matches possible — report as low-confidence region |
| **Outlier rejection** | Three-layer defense: (1) LightGlue dustbin [SuperGlue §P0], (2) DEGENSAC [LoFTR_IMC21], (3) residual Z-score filter [SIFT-IIRS-WAC] | Learned confidence threshold removes ~80% of outliers. DEGENSAC removes geometrically inconsistent matches. Z-score removes residual outliers | None significant — triple-layer is robust |
| **Spatial distribution** | ANMS/SSC [Supplementary_Research §1] + grid-based enforcement | Confidence-weighted spatial suppression guarantees uniform coverage while retaining strongest matches | Trade-off between uniformity and total match count — tunable via suppression radius |
| **Sub-pixel accuracy** | Phase-correlation paraboloid fitting [HybridPhaseCorrelation §P0] | Per-correspondence analytical sub-pixel refinement. Demonstrated RMSE < 0.2 px on satellite imagery | Fails on textureless patches (rejected, not wrong) |

---

# 6. RESEARCH INTEGRATION MAP

| Research Paper/Source | Key Finding Used | Architectural Component(s) |
|---|---|---|
| [Traditional_vs_DL](file:///c:/Workspace/code/Research/extracted/refiner/Traditional_vs_DeepLearning_FeatureMatching.md) | SuperGlue is the ONLY method succeeding across ALL 6 test conditions | Stage 3A: LightGlue as primary matcher |
| [Traditional_vs_DL](file:///c:/Workspace/code/Research/extracted/refiner/Traditional_vs_DeepLearning_FeatureMatching.md) | Heavy preprocessing doesn't rescue classical methods | Stage 2: minimal preprocessing |
| [Traditional_vs_DL](file:///c:/Workspace/code/Research/extracted/refiner/Traditional_vs_DeepLearning_FeatureMatching.md) | Preprocessing pipeline for OHRC-NAC / IIRS-WAC | Stage 2: normalization approach |
| [SuperGlue](file:///c:/Workspace/code/Research/extracted/refiner/SuperGlue.md) | Attention-based matching + dustbin outlier rejection + day-night generalization | Stage 3A: LightGlue architecture choice + confidence thresholding |
| [SuperGlue](file:///c:/Workspace/code/Research/extracted/refiner/SuperGlue.md) | Detector repeatability as bottleneck | Stage 3A: supplementary PC keypoints |
| [CNSFM](file:///c:/Workspace/code/Research/extracted/refiner/CNSFM_Crater_Neighborhood_Matching.md) | 72.3% SR at south pole via geometry-only matching | Stage 3B: extreme illumination path |
| [CNSFM](file:///c:/Workspace/code/Research/extracted/refiner/CNSFM_Crater_Neighborhood_Matching.md) | MCR outlier removal (62.2% → 100% RCM) | Stage 3B: structural outlier rejection |
| [CNSFM](file:///c:/Workspace/code/Research/extracted/refiner/CNSFM_Crater_Neighborhood_Matching.md) | Crater-sparse terrain failure | Stage 3B: fallback to Stage 3A |
| [RIFT](file:///c:/Workspace/code/Research/extracted/refiner/RIFT_extracted.md) | Phase congruency is illumination-invariant by construction | Stage 3A: supplementary PC keypoints |
| [RIFT](file:///c:/Workspace/code/Research/extracted/refiner/RIFT_extracted.md) | PC map as descriptor fails — need MIM-like encoding | Decision NOT to use PC maps as descriptors alone |
| [HybridPhaseCorrelation](file:///c:/Workspace/code/Research/extracted/refiner/HybridPhaseCorrelation%282026%29.md) | Paraboloid sub-pixel refinement achieves RMSE 0.010 px | Stage 6: sub-pixel refinement method |
| [HybridPhaseCorrelation](file:///c:/Workspace/code/Research/extracted/refiner/HybridPhaseCorrelation%282026%29.md) | Gaussian windowing > Blackman for phase correlation | Stage 6: windowing choice |
| [HybridPhaseCorrelation](file:///c:/Workspace/code/Research/extracted/refiner/HybridPhaseCorrelation%282026%29.md) | LoFTR produces out-of-domain extrapolated matches | Stage 3A: bounds-check on LightGlue output |
| [HybridPhaseCorrelation](file:///c:/Workspace/code/Research/extracted/refiner/HybridPhaseCorrelation%282026%29.md) | Terrain-slope-stratified accuracy analysis | Evaluation plan: terrain-type stratification |
| [HybridPhaseCorrelation](file:///c:/Workspace/code/Research/extracted/refiner/HybridPhaseCorrelation%282026%29.md) | Evaluation metric suite (RMSE, %<1px, %<0.5px, MedAE, inlier ratio) | Stage 7: quality metrics |
| [SIFT-IIRS-WAC](file:///c:/Workspace/code/Research/extracted/refiner/SIFT-IIRS-WAC_extracted.md) | Tiled SIFT + normalization + RANSAC + GCP declustering | Stage 2 normalization, Stage 5 Z-score filter, Stage 4 spatial enforcement |
| [SIFT-IIRS-WAC](file:///c:/Workspace/code/Research/extracted/refiner/SIFT-IIRS-WAC_extracted.md) | Solar incidence mismatch as dominant error source | Stage 1: difficulty classification |
| [SIFT-IIRS-WAC](file:///c:/Workspace/code/Research/extracted/refiner/SIFT-IIRS-WAC_extracted.md) | Latitude/curvature error amplification beyond ±55° | Evaluation plan: latitude-stratified testing |
| [DESCA](file:///c:/Workspace/code/Research/extracted/refiner/DESCA%28Dr.Sourabh%29.md) | DE optimization outperforms RANSAC-family (RMSE 0.67–0.95 px) | Stage 5: DESCA as optional upgrade to DEGENSAC |
| [DESCA](file:///c:/Workspace/code/Research/extracted/refiner/DESCA%28Dr.Sourabh%29.md) | Global affine fails on terrain relief | Stage 5: polynomial (not affine) for final warp |
| [DESCA](file:///c:/Workspace/code/Research/extracted/refiner/DESCA%28Dr.Sourabh%29.md) | M-UR-SIFT uniform distribution strategy | Stage 4: ANMS concept |
| [LoFTR_IMC21](file:///c:/Workspace/code/Research/extracted/refiner/LoFTR_IMC21.md) | DEGENSAC for degeneracy-aware verification | Stage 5: DEGENSAC choice |
| [LoFTR_IMC21](file:///c:/Workspace/code/Research/extracted/refiner/LoFTR_IMC21.md) | Adaptive NMS point merging | Stage 4: ANMS confirmation |
| [LoFTR_IMC21](file:///c:/Workspace/code/Research/extracted/refiner/LoFTR_IMC21.md) | Grid-based detection guarantees spatial coverage but lacks consistency | Decision to use detector-based (not pure grid) matching |
| [MoonMetaSync](file:///c:/Workspace/code/Research/extracted/refiner/MoonMetaSync_extracted.md) | Geodetic pre-matching for OHRC↔TMC-2 | Stage 1: metadata-driven narrowing |
| [MoonMetaSync](file:///c:/Workspace/code/Research/extracted/refiner/MoonMetaSync_extracted.md) | SIFT > ORB > IntFeat on lunar imagery | Decision NOT to use ORB as primary |
| [MoonMetaSync](file:///c:/Workspace/code/Research/extracted/refiner/MoonMetaSync_extracted.md) | Naive SIFT+ORB fusion amplifies noise | Decision NOT to use naive descriptor fusion |
| [MoonMetaSync](file:///c:/Workspace/code/Research/extracted/refiner/MoonMetaSync_extracted.md) | Bilinear vs bicubic depends on Sun angle | Stage 2: adaptive interpolation |
| [KAZE](file:///c:/Workspace/code/Research/extracted/refiner/KAZE%282026%29.md) | PC-weighted keypoint scoring improves repeatability | Stage 3A: PC-weighted supplementary detection concept |
| [KAZE](file:///c:/Workspace/code/Research/extracted/refiner/KAZE%282026%29.md) | MI + DE fine refinement achieves 0.57–0.82 px RMSE | Alternative sub-pixel method (heavier than phase correlation) |
| [KAZE](file:///c:/Workspace/code/Research/extracted/refiner/KAZE%282026%29.md) | Pipeline fails on optical-vs-SAR (large modality gap) | IIRS handling as separate module |
| [NASA_SubPixel](file:///c:/Workspace/code/Research/extracted/refiner/NASA_SubPixel_Refinment%281d%29.md) | Coarse-to-fine grid search + sub-pixel refinement pattern | Stage 6: coarse-to-fine design principle |
| [NASA_SubPixel](file:///c:/Workspace/code/Research/extracted/refiner/NASA_SubPixel_Refinment%281d%29.md) | Sensor footprint/pixel ratio governs achievable accuracy | Per-sensor-pair accuracy expectations |
| [Radiometric_Normalization](file:///c:/Workspace/code/Research/extracted/refiner/Radiometric_Normalization_Analysis.md) | Histogram/Retinex normalization insufficient for nonlinear illumination | Stage 2: no reliance on simple normalization as illumination fix |
| [Supplementary_Research](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/SIH26166_supplementary_research%282%29.md) | ANMS/SSC for uniform distribution | Stage 4: ANMS |
| [Supplementary_Research](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/SIH26166_supplementary_research%282%29.md) | ASP + ISIS3 as infrastructure | Stage 2: calibration |
| [Additional_Resources](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/Additional_Resources_SIH26166.md) | LightGlue as SuperGlue successor, hloc toolbox | Stage 3A: implementation base |
| [Additional_Resources](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/Additional_Resources_SIH26166.md) | DeepMoon + TMC-2 crater U-Net | Stage 3B: crater detector options |
| [Additional_Resources](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/Additional_Resources_SIH26166.md) | Synthetic lunar image generation tools | Training data generation for fine-tuning |
| [Reference_Mapping](file:///c:/Workspace/code/Research/extracted/refiner/ai_suggestions/SIH26166_reference_mapping.md) | LROC WAC/NAC as trusted reference frame | Stage 1: reference image selection |

---

# 7. WHY THESE METHODS WORK TOGETHER

## LightGlue + Phase Congruency Keypoints

**Technical interface:** LightGlue accepts any set of keypoints with associated descriptors. SuperPoint provides the primary keypoints (repeatable on textured surfaces under moderate conditions). PC keypoints fill detection gaps in low-texture and illumination-extreme areas where SuperPoint's gradient-based detection fails. The combined keypoint set gives LightGlue more candidates to reason about, and its attention mechanism can attend to PC-sourced keypoints that have different spatial characteristics than SuperPoint's.

**Why one compensates for the other:** SuperPoint was trained on natural images — its detection heatmap may not fire on lunar-specific structures (subtle crater rims, regolith texture boundaries). PC detection is domain-agnostic (frequency-domain phase alignment) and is specifically demonstrated as illumination-invariant on cross-sensor pairs [RIFT, KAZE]. LightGlue's cross-attention can compare these PC keypoints against both SuperPoint-detected and PC-detected keypoints in the other image, effectively extending LightGlue's reach into illumination-challenging regions.

## LightGlue + CNSFM

**Technical interface:** These operate on **different information channels** entirely. LightGlue operates on pixel appearance (learned features). CNSFM operates on geometric topology (crater positions). They are fundamentally non-redundant.

**Why one compensates for the other:** Under extreme illumination change (>60° azimuth difference), pixel appearance changes nonlinearly due to shadow boundary migration [CNSFM §P2]. LightGlue may still succeed (it generalizes across day↔night [SuperGlue]), but if it fails, CNSFM provides an independent pathway that does not use pixel intensity at any stage. Conversely, in crater-sparse terrain, CNSFM has zero matches — but these regions are exactly where LightGlue's learned texture matching works (smooth mare has moderate, uniform texture that LightGlue handles via global context).

## Feature Matching + Phase-Correlation Sub-Pixel Refinement

**Technical interface:** Feature matching provides integer/near-integer-pixel correspondence locations. Phase correlation operates on small patches around these locations to compute sub-pixel correction vectors. The two are sequential, not redundant.

**Why one compensates for the other:** [HybridPhaseCorrelation §P2] establishes that "sub-pixel refinement strategy choice matters more than raw detector choice, but the two interact." LightGlue's sub-pixel capability is limited to its coarse-to-fine learned refinement (~0.4–0.9 px RMSE). Phase-correlation paraboloid fitting achieves RMSE <0.2 px [HybridPhaseCorrelation §P0] — a full order-of-magnitude improvement. The feature matcher provides **where** to look; phase correlation provides **exactly where** to align.

## Spatial Distribution Enforcement + DEGENSAC

**Technical interface:** ANMS runs before DEGENSAC, ensuring DEGENSAC operates on a spatially uniform point set rather than a cluster. This prevents DEGENSAC from fitting a model dominated by a spatially concentrated group of matches (which would bias the geometric estimate toward one image region).

**Why one compensates for the other:** Without spatial enforcement, DEGENSAC may find a geometrically consistent model that only explains matches in one high-texture area (e.g., a single large crater cluster) while ignoring misalignment in other regions. ANMS forces matches to be distributed, so DEGENSAC's model must explain geometric consistency across the entire image.

---

# 8. WHAT WE DELIBERATELY EXCLUDED

| Method | Why It Looked Attractive | Why Excluded | May Be Useful Later? |
|--------|-------------------------|--------------|---------------------|
| **RIFT/RIFT2 as primary matcher** | Illumination-invariant by design; no training data needed [RIFT] | Failed on OHRC-NAC Polar and DFSAR-SELENE [Traditional_vs_DL]; RMSE 1.5/1.2 px (vs LightGlue 0.4–0.9 px); lacks scale invariance [RIFT §P1] | Yes — as a lightweight no-GPU fallback, or as a source of PC+MIM supplementary features |
| **LNIFT as primary method** | 100× faster than RIFT, 99.9% SR, built-in ANMS [Supplementary_Research §3] | Not benchmarked on the exact Chandrayaan-2 sensor pairs or lunar polar conditions; evidence is on Earth remote sensing only; less comprehensive validation than LightGlue/SuperGlue | Yes — strong candidate for speed-critical deployment; worth benchmarking as a comparison baseline |
| **LoFTR (detector-free)** | Sub-pixel, semi-dense, uniform grid coverage [LoFTR_IMC21] | Produces out-of-domain extrapolated matches [HybridPhaseCorrelation §P4]; pure grid-based detection lacks repeatability across views [LoFTR_IMC21 §P4]; superseded by LightGlue in the same research lineage | No — LightGlue is the direct successor |
| **DESCA (DE-based outlier rejection)** | Sub-pixel RMSE 0.67–0.95 px [DESCA] | Only validated on optical-optical same-modality; only with affine model (fails on terrain relief [DESCA §P4]); requires careful two-tier initialization [DESCA §P2]; adds significant compute (200 DE generations) | Yes — as a DEGENSAC replacement in Phase 3 if accuracy demands it |
| **cGAN radiometric normalization** | Learned nonlinear illumination mapping [Radiometric_Normalization] | Assumes pre-registered images (the opposite of our problem); requires paired training data that doesn't exist for our sensors; not validated for preserving fine structural cues needed for matching | Unlikely — wrong problem formulation |
| **MI + DE sub-pixel refinement** [KAZE] | RMSE 0.57–0.82 px, statistically significant vs SPSA | Heavier compute than phase correlation; only validated with affine model; parameter tuning (N=20, S=0.8) is dataset-specific | Yes — as alternative to phase correlation if paraboloid fitting proves insufficient |
| **HOPC** [RIFT] | Multi-modal template matching | "Completely unusable without accurate geolocation" [RIFT §P4]; our metadata may be unreliable | No |
| **IntFeat (SIFT+ORB fusion)** | Combine SIFT richness + ORB speed [MoonMetaSync] | "Did not exceed SIFT's accuracy in any tested condition"; ORB features amplify noise [MoonMetaSync §P4] | No |
| **PointCN/OANet outlier classifiers** | Learned filtering of NN matches [SuperGlue §P4] | "Cannot predict more correct matches than the NN matcher itself" — fundamentally bounded by NN recall | No — confirmed dead end |
| **Simoncelli pyramids** | More sophisticated multi-scale representation [HybridPhaseCorrelation] | "Need careful tuning and more computation for uncertain benefit" | Unlikely |
| **RoMa/MatchAnything** | Latest dense matchers with extreme illumination robustness [Additional_Resources §6-7] | Not analyzed in full P0-P6 format; insufficient evidence in our research base to make architectural decisions. Promising but unvalidated for lunar imagery | Yes — HIGH PRIORITY for future investigation; could replace LightGlue if cross-modal performance proves superior |

---

# 9. FAILURE MODES AND FALLBACKS

## Scenario 1: LightGlue produces <15 matches

**Cause:** Low-texture region (mare), extreme shadow, or large domain gap.

**Detection:** Match count threshold after Stage 3A.

**Fallback:**
1. Activate Stage 3B (CNSFM) if solar-angle difference is large.
2. If CNSFM also fails (crater-sparse), relax LightGlue confidence threshold from 0.2 → 0.1 and re-run.
3. If still insufficient: flag image pair as "LOW CONFIDENCE — manual review recommended."

## Scenario 2: DEGENSAC finds no consistent model

**Cause:** All matches are outliers, or geometric distortion is too complex for the chosen model.

**Detection:** DEGENSAC returns <4 inliers.

**Fallback:**
1. Reduce model complexity: try similarity (4-param) instead of affine (6-param).
2. If still failing: try per-tile estimation with smaller tiles (allowing local models).
3. Final fallback: report failure with diagnostic information.

## Scenario 3: Sub-pixel refinement degrades accuracy

**Cause:** Phase correlation on textureless patches produces spurious peaks.

**Detection:** Refinement shift > 2 px or correlation peak < threshold.

**Fallback:** Reject refinement for that specific match; retain the feature-matching-level position. Report the un-refined match with a lower confidence score.

## Scenario 4: Crater detector fails (Stage 3B)

**Cause:** Imagery is from crater-sparse terrain, or detector is not calibrated for this sensor.

**Detection:** <10 craters detected.

**Fallback:** Skip Stage 3B entirely; rely on Stage 3A with relaxed thresholds.

## Scenario 5: Extreme resolution mismatch (OHRC 0.25m ↔ IIRS 80m)

**Cause:** 320× resolution ratio destroys texture during resampling.

**Detection:** Resolution ratio > 10× (computed in Stage 1).

**Handling:** Treat IIRS as a separate module: use a band-selection step to pick the most texture-rich IIRS band, match against LRO WAC (comparable ~100m resolution) rather than directly against OHRC, then chain transformations (IIRS→WAC→OHRC via transitive registration).

---

# 10. ALTERNATIVE ARCHITECTURES

## Architecture A — Highest Expected Accuracy (RECOMMENDED)

The full pipeline described above: LightGlue + CNSFM conditional + ANMS + DEGENSAC + phase-correlation sub-pixel + per-match confidence.

**Trade-off:** Highest accuracy ceiling. Requires GPU. Moderate complexity. LightGlue fine-tuning recommended.

## Architecture B — No Training Required (Classical Hybrid)

Replace LightGlue with **RIFT2 + PC-weighted KAZE** in a coarse-to-fine strategy:
1. I-KAZE (PC-weighted) for coarse alignment [KAZE §P0]
2. RIFT2 for fine correspondence [RIFT, Traditional_vs_DL]
3. CNSFM for extreme illumination (same as Architecture A)
4. ANMS for spatial distribution (same)
5. RANSAC (standard, not DEGENSAC) for geometric estimation
6. MI + DE for sub-pixel refinement [KAZE §P0]

**Trade-off:** No training data needed. No GPU required for inference. Lower accuracy ceiling (RIFT2 RMSE ~1.5 px vs LightGlue ~0.5 px). Fails on polar OHRC-NAC and SAR-optical [Traditional_vs_DL]. Heavier compute than LightGlue.

**When to choose:** If GPU is unavailable AND training data cannot be generated AND polar/SAR cases are not in scope.

## Architecture C — Simplest Viable System

Minimal pipeline for fastest implementation:
1. Metadata bounding-box crop (Stage 1)
2. Percentile-normalization (Stage 2, simplified)
3. SIFT detection + brute-force matching + Lowe ratio test
4. RANSAC homography
5. GCP declustering [SIFT-IIRS-WAC]
6. OpenCV `cornerSubPix` for sub-pixel refinement

**Trade-off:** Implementable in one week. Uses only OpenCV. Fails on polar, cross-modal, and extreme illumination cases [demonstrated across all papers]. RMSE 1–6 px. Does not meet the problem statement's robustness requirements.

**When to choose:** Only as a Phase 1 baseline to benchmark against — NOT as the final deliverable.

---

# 11. RECOMMENDED ARCHITECTURE

## **Architecture A is recommended.**

**Justification by priority criteria:**

1. **ACCURACY:** LightGlue achieves the lowest demonstrated RMSE (~0.4–0.9 px) on our actual sensor pairs [Traditional_vs_DL]. Phase-correlation refinement pushes this toward <0.2 px [HybridPhaseCorrelation]. No other architecture in our alternatives matches this.

2. **ROBUSTNESS:** LightGlue is the only method that succeeded on ALL 6 test conditions (including polar and cross-modal) [Traditional_vs_DL]. The CNSFM fallback extends robustness to the most extreme illumination cases where even LightGlue may struggle.

3. **FEASIBILITY:** LightGlue + SuperPoint are available as Apache-2.0 open-source code via `cvg/LightGlue`. DEGENSAC is available in pydegensac. ANMS is available at `BAILOOL/ANMS-Codes`. CNSFM is available at `Bin501/CNSFM`. Phase correlation is implementable in <100 lines of numpy/scipy. The `hloc` toolbox [Additional_Resources §10] already wires SuperPoint + LightGlue + COLMAP together. A student team can build on this existing infrastructure rather than starting from scratch.

4. **COMPLEXITY:** Seven stages, but each is independently testable and has a clear, justified purpose. No stage duplicates another's function. The conditional CNSFM path adds one branch, not a parallel system.

---

# 12. ABLATION STUDY

| # | Configuration | Component Removed | Expected Effect | Metric Affected | What It Proves |
|---|---|---|---|---|---|
| 1 | **Full system** | — | Baseline | All | — |
| 2 | Remove CNSFM (Stage 3B) | Extreme illumination path | Moderate accuracy drop on polar/high-azimuth pairs; no effect on equatorial | Success rate on polar test set; RMSE on >60° azimuth pairs | Whether CNSFM adds value beyond what LightGlue already handles |
| 3 | Remove PC supplementary keypoints | Phase congruency detection | Small accuracy drop in low-texture and illumination-extreme regions | Match count in mare regions; inlier ratio under illumination change | Whether PC keypoints extend LightGlue's reach |
| 4 | Remove ANMS (Stage 4) | Spatial distribution enforcement | No RMSE change but poor spatial coverage; matches cluster in high-texture regions | Spatial distribution metric (grid density std-dev) | Whether ANMS is needed for uniform coverage |
| 5 | Replace DEGENSAC with standard RANSAC | Degeneracy awareness | Possible model degeneracy on flat mare; slight accuracy drop | RMSE on flat-terrain test pairs; failure rate on mare scenes | Whether DEGENSAC's degeneracy handling matters for lunar terrain |
| 6 | Remove sub-pixel refinement (Stage 6) | Phase-correlation paraboloid fitting | RMSE increases from <0.5 px to ~0.5–0.9 px | RMSE; %<0.5 px; MedAE | Quantifies sub-pixel refinement contribution |
| 7 | Remove Z-score residual filter | Final outlier cleaning | Small increase in outlier contamination | Inlier ratio; RMSE (slight increase due to residual outliers) | Whether triple-layer filtering is better than double-layer |
| 8 | Replace LightGlue with RIFT2 | Learned matcher | Large accuracy drop on polar/cross-modal; no training data needed | RMSE; success rate on polar; success rate on cross-modal | Quantifies the value of learned vs. classical matching |
| 9 | Remove metadata narrowing (Stage 1) | Spatial search reduction | Massive increase in false matches; much higher compute | Runtime; false positive rate; RMSE | Whether metadata pre-processing is necessary |
| 10 | Full system with DESCA replacing DEGENSAC | Swap robust estimator | Possible RMSE improvement from 0.5 → 0.3–0.5 px; higher compute | RMSE; runtime | Whether DE optimization adds accuracy vs. DEGENSAC |

---

# 13. EVALUATION PLAN

## Metrics

| Metric | Formula/Definition | Target |
|--------|-------------------|--------|
| **RMSE** | √(Σ(error²)/N) in pixels, measured against held-out checkpoints | < 1.0 px for equatorial same-sensor; < 2.0 px for cross-sensor; < 0.5 px after sub-pixel refinement |
| **Sub-pixel accuracy %** | % of correspondences with error < 1.0 px | > 80% |
| **High-precision accuracy %** | % of correspondences with error < 0.5 px | > 50% |
| **MedAE** | Median absolute error | < 0.5 px |
| **Inlier count** | Number of matches surviving DEGENSAC + Z-score | > 50 per image pair |
| **Inlier ratio** | Inliers / total proposed matches | > 0.5 (higher = cleaner matcher) |
| **Spatial distribution** | Std-dev of match density across N×N grid cells | < 2× mean (uniformity measure) |
| **Registration success rate** | % of image pairs where RMSE < threshold | > 95% equatorial; > 70% polar |
| **Runtime** | End-to-end processing time per image pair | < 30 seconds on GPU |

## Test Categories

| Category | Description | Target Sensor Pairs | Challenge Tested |
|----------|-------------|--------------------|-----------------| 
| **EQ-SS** | Equatorial, same-sensor | OHRC↔OHRC, TMC↔TMC | Baseline accuracy |
| **EQ-XS** | Equatorial, cross-sensor | OHRC↔LRO NAC, TMC↔LRO WAC | Multi-modal, scale |
| **EQ-XS-WIDE** | Equatorial, cross-sensor, large scale gap | OHRC↔TMC (17×) | Extreme scale |
| **POL-SS** | Polar (>60° lat), same-sensor | OHRC↔OHRC polar | Extreme illumination |
| **POL-XS** | Polar, cross-sensor | OHRC↔LRO NAC polar | Illumination + multi-modal |
| **SUN-LOW** | Low Sun elevation (<20°) | Any pair | Shadow dominance |
| **SUN-HIGH** | High Sun elevation (>60°) | Any pair | Low shadow contrast |
| **AZI-DIFF** | Large azimuth difference (>90°) | Any pair | Shadow boundary migration |
| **LOW-TEX** | Mare / low-texture terrain | Any pair | Feature detection failure |
| **HIGH-REL** | Highland / high-relief terrain | Any pair | Local geometric distortion |
| **IIRS** | Hyperspectral cross-modal | IIRS↔LRO WAC | Spectral + resolution gap |

## Ground Truth Strategy

1. **Synthetic transforms** (Phase 1): Apply known geometric transforms to real lunar images. Exact ground truth. Good for pipeline validation.
2. **Held-out checkpoints** (Phase 2): From matched points, hold out 20% as checkpoints (not used for model fitting). Measure error on checkpoints only. [Supplementary_Research §6].
3. **Cross-reference against ASP** (Phase 3): Compare our correspondence output against NASA ASP's own interest-point matching + bundle adjustment results on the same OHRC/TMC data. ASP serves as an independent reference pipeline.
4. **LRO NAC stereo DEM as reference** (Phase 3): For OHRC↔NAC pairs, existing stereo DEMs [Additional_Resources §1-2] provide independent geometric ground truth with ~30 m horizontal / 8 cm vertical accuracy.

---

# 14. IMPLEMENTATION ROADMAP

## Phase 1 — Baseline (Weeks 1–2)

**Goal:** Working end-to-end pipeline on synthetic data.

- Set up ISIS3 + ASP for data ingestion and calibration.
- Implement Stage 1 (metadata bounding-box crop).
- Implement Stage 2 (percentile normalization, resolution matching).
- Implement Architecture C (SIFT + RANSAC + cornerSubPix) as the baseline.
- Build evaluation harness (RMSE, inlier ratio, spatial distribution metric).
- Test on synthetic transforms of real OHRC/TMC images.
- **Deliverable:** Baseline RMSE numbers. Working evaluation framework.

## Phase 2 — Core Matching System (Weeks 3–4)

**Goal:** LightGlue integration and primary matching pipeline.

- Integrate SuperPoint + LightGlue via `hloc` toolbox.
- Replace SIFT in the baseline with LightGlue.
- Implement DEGENSAC (replace standard RANSAC).
- Implement ANMS spatial distribution enforcement.
- Test on real OHRC↔LRO NAC and TMC↔LRO WAC pairs.
- Compare against Phase 1 baseline.
- **Deliverable:** LightGlue-based RMSE on real data vs. SIFT baseline.

## Phase 3 — Robustness Extensions (Weeks 5–6)

**Goal:** Handle extreme conditions.

- Implement Stage 3B: CNSFM conditional path.
  - Integrate crater detector (DeepMoon or fine-tuned YOLOv9).
  - Implement CNSFM geometric matching + MCR outlier removal.
- Implement supplementary PC keypoint extraction feeding into LightGlue.
- Implement difficulty classification (solar-angle routing).
- Test on polar and high-azimuth-difference pairs.
- **Deliverable:** Success rate on polar test set. Ablation: with vs. without CNSFM.

## Phase 4 — Sub-Pixel Refinement (Week 7)

**Goal:** Achieve sub-pixel accuracy.

- Implement Stage 6: phase-correlation paraboloid refinement.
- Implement per-match confidence scoring (Stage 7).
- Implement Z-score residual filter.
- Measure RMSE improvement from refinement.
- **Deliverable:** Sub-pixel RMSE numbers. Ablation: with vs. without refinement.

## Phase 5 — Spatial Distribution & IIRS (Week 8)

**Goal:** Uniform coverage + IIRS handling.

- Tune ANMS parameters for optimal coverage vs. count trade-off.
- Implement IIRS module (band selection → match against WAC → transitive registration).
- Measure spatial distribution metric across test categories.
- **Deliverable:** Spatial coverage numbers. IIRS registration results.

## Phase 6 — Evaluation & Ablation (Weeks 9–10)

**Goal:** Rigorous evaluation and final report.

- Run full ablation study (10 configurations × all test categories).
- Run terrain-stratified accuracy analysis (mare vs. highland vs. crater-rim).
- Run latitude-stratified analysis.
- Compare against ASP as independent reference.
- Generate registered output products.
- Write quality report with all metrics.
- **Deliverable:** Complete evaluation results. Registered image products. Final report.

---

# 15. FINAL TECHNICAL SPECIFICATION

```
INPUT: Chandrayaan-2 image (OHRC / TMC / IIRS) 
       + Reference image (LRO WAC mosaic / LRO NAC strip / SELENE TC)
       + Spacecraft metadata (corner coords, solar angles)

→ STAGE 1 [NARROW]: Geodetic bounding-box crop (pygeodesy / GDAL)
                     Solar-angle difference → difficulty flag
                     Resolution ratio → resampling strategy

→ STAGE 2 [PREP]:   ISIS3 calibration (if raw PDS4)
                     Bilinear/bicubic resampling (Sun-angle-adaptive)
                     Percentile-clip normalization (2nd/98th → [0,1] → uint8)

→ STAGE 3 [MATCH]:  
    PRIMARY PATH (≤60° azimuth diff):
      SuperPoint detection (pretrained + optional lunar fine-tune)
      + Phase Congruency supplementary keypoints (log-Gabor, No=6, Ns=4)
      → LightGlue matching (Apache-2.0, via hloc)
      → Confidence threshold (dustbin > 0.2)
      → Bounds-validation check
    
    EXTREME PATH (>60° azimuth diff OR primary <15 matches):
      YOLOv9 / DeepMoon crater detection
      → CNSFM geometric topology matching (K-NN structure, similarity invariants)
      → MCR outlier removal
      → FALLBACK to primary if <10 craters

→ STAGE 4 [DISTRIBUTE]: ANMS-SSC (suppression-via-square-covering)
                         Confidence-weighted, adaptive radius
                         Floor: minimum 30 matches retained

→ STAGE 5 [ESTIMATE]:   DEGENSAC (0.5 px threshold, 10K iterations, 0.99999 conf)
                         Similarity → Affine → Polynomial (progressive complexity)
                         Z-score residual filter (>2.5σ removal, requires >20 inliers)

→ STAGE 6 [REFINE]:     Per-match 64×64 patch extraction
                         Gaussian-windowed phase-only cross-correlation
                         3×3 2D paraboloid peak fitting → sub-pixel displacement
                         Reject if shift >2px OR correlation <0.3

→ STAGE 7 [VALIDATE]:   Composite confidence scoring
                         One-to-one enforcement
                         Quality metrics computation

→ OUTPUT: Correspondence set [(x_src, y_src, x_ref, y_ref, confidence)]
          Geometric transformation (polynomial coefficients)
          Registered/warped image in reference CRS
          Quality report: RMSE, %<1px, %<0.5px, MedAE, 
                          inlier count, inlier ratio,
                          spatial distribution metric,
                          registration success/failure flag
```

> [!IMPORTANT]
> **Key proposed integrations (not yet demonstrated in literature, require validation):**
> 1. PC keypoints as supplementary input to LightGlue
> 2. CNSFM as conditional fallback activated by solar-angle metadata
> 3. Phase-correlation sub-pixel refinement applied to LightGlue's output correspondences
>
> All three are architecturally sound but must be empirically validated in Phase 3–4 of implementation.

---

*Architecture designed from evidence base of 13 research extractions spanning CNSFM, DESCA, Hybrid Phase Correlation, I-KAZE, LoFTR, MoonMetaSync, NASA Sub-Pixel Refinement, RIFT, Radiometric Normalization, SIFT-IIRS-WAC, SuperGlue, Traditional vs Deep Learning Feature Matching, and supplementary research resources.*
