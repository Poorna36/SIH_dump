# ARCHITECTURE COMPANION DOCUMENT
## Technical Decision Records, Interface Specifications & Risk Analysis
### SIH 26166 — Lunar Multi-Modal Image Correspondence

---

# 1. COMPONENT CLASSIFICATION MATRIX

Every architectural component is classified by its contribution to the system's accuracy, with evidence-based justification for why it occupies that tier.

## Classification Definitions

| Tier | Meaning | Design Rule |
|------|---------|-------------|
| **ESSENTIAL** | Removing it causes the system to fail on one or more stated requirements | Must be in the final system. No negotiation. |
| **HIGH-VALUE** | Strong evidence of material accuracy improvement; system works without it but measurably worse | Include unless implementation cost is prohibitive. Validate via ablation. |
| **OPTIONAL** | Useful under specific conditions; system works without it in most cases | Include if time permits. Clearly scoped to specific conditions. |
| **EXPERIMENTAL** | Promising idea from research but not validated in this specific context | Mark as [PROPOSED INTEGRATION]. Do not depend on it for core accuracy. |
| **UNNECESSARY** | Looked attractive but either redundant or insufficient evidence | Exclude. Document why. |

---

## Classification Table

| Component | Tier | Justification | What Happens If Removed |
|-----------|------|---------------|------------------------|
| **Stage 1: Metadata spatial narrowing** | ESSENTIAL | Without it, the system searches the entire reference mosaic — infeasible at OHRC 0.25m resolution. [MoonMetaSync] demonstrated this on OHRC↔TMC-2. [Reference_Mapping] confirms metadata→patch→matching is the standard lunar workflow. | System becomes computationally infeasible. Massive false-match rate from searching entire Moon. |
| **Stage 2: Percentile-clip normalization** | ESSENTIAL | Cross-sensor radiometric differences (OHRC vs NAC, IIRS vs WAC) require intensity harmonization before any matching. [SIFT-IIRS-WAC] used this exact normalization on Chandrayaan-2 IIRS. Without it, even LightGlue operates on mismatched intensity distributions. | Cross-sensor matching accuracy drops. Learned features may not transfer well across raw intensity distributions. |
| **Stage 2: Resolution resampling** | ESSENTIAL | OHRC (0.25m) vs TMC (5m) is a 20× resolution gap. Feature detectors cannot match features across fundamentally different scales without bringing images to comparable resolution. [MoonMetaSync] demonstrated this on the exact OHRC↔TMC-2 pair. | Feature detection finds incomparable structures at different scales. Matching fails. |
| **Stage 2: Adaptive interpolation (bilinear/bicubic by Sun angle)** | OPTIONAL | [MoonMetaSync §P2] showed bicubic amplifies shadow artifacts at low Sun angles. Effect is real but modest — affects interpolation quality, not whether matching works at all. | Slight quality loss in resampled imagery under low Sun angles. Not fatal. |
| **Stage 3A: LightGlue as primary matcher** | ESSENTIAL | Only SuperGlue/LightGlue succeeded across ALL 6 test conditions on our exact sensors [Traditional_vs_DL]. Every alternative failed on at least one condition (polar, cross-modal). This is the single strongest evidence point in the entire research base. | System fails on polar imagery and cross-modal pairs. Falls back to RIFT2/SIFT accuracy regime (1.5–6 px RMSE instead of 0.4–0.9 px). |
| **Stage 3A: SuperPoint as front-end detector** | ESSENTIAL (for LightGlue) | LightGlue is co-designed with SuperPoint. Best performance demonstrated with this pairing [SuperGlue §P1]. LightGlue is front-end agnostic in principle, but SuperPoint is the validated default. | Must use an alternative detector. May lose the co-optimized performance that makes LightGlue effective. |
| **Stage 3A: Supplementary PC keypoints** | EXPERIMENTAL | [PROPOSED INTEGRATION] — PC keypoints feeding into LightGlue has never been demonstrated. Rationale is sound (PC is illumination-invariant [RIFT], LightGlue is front-end agnostic [SuperGlue §P1]), but outcome is unknown. | Probably minor impact in most conditions. May matter in extreme illumination cases where SuperPoint detection degrades. Validate in ablation. |
| **Stage 3B: CNSFM crater-geometry matching** | HIGH-VALUE | 72.3% success at south pole where best alternative hit 31.2% [CNSFM §P0]. This is the only method with strong evidence for the hardest illumination cases. But it fails in crater-sparse terrain, has lower RMSE (1–2.2 px vs 0.5 px), and produces sparse matches (tens, not hundreds). | System loses its best fallback for extreme polar/azimuth-difference cases. LightGlue may still handle many of these — but the evidence gap is that LightGlue was tested on Chandrayaan-2 polar with *generic* weights, not lunar-fine-tuned weights, so its polar ceiling is unknown. |
| **Stage 3B: MCR outlier removal** | HIGH-VALUE (if using CNSFM) | Ablation in [CNSFM §P1] shows MCR alone raises RCM from 62.2% → 100%. Massive demonstrated contribution to CNSFM reliability. But only relevant if CNSFM is active. | If CNSFM is used without MCR, match reliability drops to ~62%. |
| **Stage 4: ANMS spatial distribution** | ESSENTIAL | Problem statement *explicitly* requires "UNIFORM SPATIAL DISTRIBUTION of reliable match points." No matcher inherently provides this — they cluster matches in high-texture regions (crater clusters). [Supplementary_Research §1] identifies ANMS/SSC as the standard solution. [LoFTR_IMC21] confirms the pattern. | Matches cluster in high-texture areas. Registration model biased toward those regions. The explicit PS requirement for uniform distribution is unmet. |
| **Stage 5: DEGENSAC robust estimation** | ESSENTIAL | Outlier rejection is an explicit PS requirement. DEGENSAC adds degeneracy awareness over standard RANSAC — important because lunar mare is large, flat, near-planar terrain where standard RANSAC can produce degenerate models [LoFTR_IMC21 §P0]. | Without any robust estimation: catastrophic outlier contamination. Without degeneracy awareness (plain RANSAC instead): risk of degenerate models on flat terrain. |
| **Stage 5: Z-score residual filter** | HIGH-VALUE | [SIFT-IIRS-WAC §P0] uses this operationally on Chandrayaan-2 data. Catches residual outliers that survive RANSAC — third defense layer. Low cost. | Slight increase in outlier contamination. DEGENSAC handles most outliers already. Z-score is a safety net. |
| **Stage 5: Polynomial warp (not pure homography)** | HIGH-VALUE | [DESCA §P4] demonstrated global affine failure on terrain relief. [SIFT-IIRS-WAC §P1] decouples RANSAC verification (homography per tile) from final warp (polynomial). Lunar craters and mountains have local relief that a single homography can't capture. | Homography works for flat terrain. Fails on rugged terrain (crater rims, highlands). For equatorial mare, probably fine. For highlands/polar, accuracy degrades. |
| **Stage 6: Phase-correlation sub-pixel refinement** | ESSENTIAL | PS explicitly requires sub-pixel accuracy. LightGlue achieves ~0.5–0.9 px RMSE [Traditional_vs_DL] — not consistently sub-pixel. [HybridPhaseCorrelation §P0] demonstrated RMSE < 0.2 px with paraboloid fitting — an order of magnitude improvement. This is the only component that bridges the gap between "good feature matching" and "sub-pixel registration." | System stops at ~0.5–0.9 px RMSE. Sub-pixel accuracy requirement is not reliably met. |
| **Stage 6: Gaussian windowing** | HIGH-VALUE | [HybridPhaseCorrelation §P4]: "Blackman performs worst because it suppresses too much high-frequency information." Gaussian/Tukey are the evidence-backed choices. Wrong windowing measurably degrades phase correlation quality. | If Blackman is used instead: accuracy loss. If no windowing: spectral leakage. Gaussian is the validated default. |
| **Stage 7: Composite confidence scoring** | OPTIONAL | No paper directly validates composite confidence scoring for lunar imagery. It's standard engineering practice but adds modest complexity. Individual confidence signals (LightGlue score, RANSAC residual, correlation peak) are each useful independently. | Users lose per-match confidence information. System still produces correspondences. |
| **Stage 7: One-to-one enforcement** | HIGH-VALUE | [LoFTR_IMC21 §P0] documents that many-to-one matches from point merging create ambiguity. One-to-one is required for clean RMSE reporting and downstream use. | Ambiguous correspondences where multiple source points map to the same reference location. Inflated match counts. |
| **Difficulty classification / routing** | OPTIONAL | Automatically routing to CNSFM vs LightGlue based on solar-angle metadata is an efficiency optimization. In the fallback design, if LightGlue produces <15 matches, CNSFM activates anyway. The metadata-based routing just avoids wasting time on LightGlue when it's likely to fail. | System always tries LightGlue first, falls back if needed. Slightly higher compute on extreme-illumination pairs. No accuracy impact — just efficiency. |
| **ISIS3 calibration** | ESSENTIAL (for raw data) | Raw PDS4 products are not geometrically or radiometrically calibrated. ISIS3 converts them to usable imagery with camera geometry. [Supplementary_Research §2] and [Additional_Resources §9] confirm this is the community-standard prerequisite. | Cannot process raw OHRC/TMC/IIRS products. Blocked on data ingestion. |
| **Bounds-validation on LightGlue output** | HIGH-VALUE | [HybridPhaseCorrelation §P4] reports LoFTR (same architecture family) produces "out-of-image-domain" matches. LightGlue may exhibit similar behavior. Simple check. | Rare but catastrophic: geometrically impossible matches contaminate the pipeline. |

---

# 2. FORMAL METHOD-SELECTION DECISION RECORDS

## Decision 1: Primary Feature Matcher

**Decision:** LightGlue (with SuperPoint front end)

| Criterion (weight) | LightGlue/SuperGlue | RIFT2 | SIFT | LNIFT | LoFTR |
|---|---|---|---|---|---|
| Polar success (30%) | ✅ Only method succeeding ALL 6 conditions [Traditional_vs_DL] | ❌ Failed OHRC-NAC Polar [Traditional_vs_DL] | ❌ Failed polar [Traditional_vs_DL, CNSFM] | ⚠️ Untested on lunar polar | ⚠️ Produces out-of-domain matches [HybridPhaseCorrelation] |
| Cross-modal (20%) | ✅ Succeeded DFSAR-SELENE [Traditional_vs_DL] | ❌ Failed DFSAR-SELENE Eq [Traditional_vs_DL] | ❌ Failed SAR [Traditional_vs_DL] | ⚠️ Untested cross-modal lunar | ⚠️ Trained on MegaDepth only [LoFTR_IMC21] |
| RMSE accuracy (20%) | ✅ 0.4–0.9 px [Traditional_vs_DL] | ⚠️ 1.5/1.2 px [Traditional_vs_DL] | ❌ 3.6/5.9 px [Traditional_vs_DL] | ⚠️ Reported fast but RMSE not on lunar | ⚠️ Competitive but unstable [HybridPhaseCorrelation] |
| Runtime (10%) | ✅ Single-digit seconds [Traditional_vs_DL] | ⚠️ 36.9s [Traditional_vs_DL] | ❌ 678–809s [Traditional_vs_DL] | ✅ 0.49s [Supplementary_Research] | ⚠️ Moderate |
| Training data need (10%) | ⚠️ Needs domain-specific fine-tuning | ✅ None | ✅ None | ✅ None | ⚠️ Needs retraining |
| Pretrained availability (10%) | ✅ Apache-2.0, hloc integration | ✅ Public | ✅ OpenCV | ✅ Public exe | ✅ Public |
| **Weighted score** | **HIGHEST** | Medium | Low | Medium-High (uncertain) | Medium |

**Why LightGlue over LNIFT (the supplementary research recommendation):**
LNIFT has compelling speed (100× RIFT) and built-in ANMS, and the supplementary research recommended it as primary. However:
1. LNIFT has **zero demonstrated results on lunar polar imagery** or cross-modal Chandrayaan-2 pairs. Its evidence base is entirely Earth remote sensing.
2. LightGlue/SuperGlue has **directly demonstrated** results on OHRC-NAC polar, IIRS-WAC polar, and DFSAR-SELENE [Traditional_vs_DL] — our exact sensors and conditions.
3. For an accuracy-first design, choosing the method with the strongest evidence on our actual problem conditions takes priority over a faster method with untested lunar performance.

**LNIFT recommendation:** Position as a **comparison baseline** in evaluation, not as the primary method. If LNIFT proves competitive on lunar data during evaluation, it becomes an attractive lightweight deployment option.

---

## Decision 2: Extreme Illumination Handler

**Decision:** CNSFM (conditional activation)

| Criterion | CNSFM | WSSF | HAPCG | ML-HLMO | No fallback (LightGlue only) |
|---|---|---|---|---|---|
| Polar SR | ✅ 72.3% [CNSFM] | ⚠️ 31.2% [CNSFM] | ❌ 22.1% [CNSFM] | ❌ 13.9% [CNSFM] | ⚠️ Unknown — LightGlue tested with generic weights [Traditional_vs_DL] |
| Illumination invariance mechanism | ✅ Geometry-only — invariant *by construction* | ⚠️ Partial structural info | ⚠️ Frequency domain — still degrades | ⚠️ Histogram-based — unstable | ✅ Learned — may generalize |
| Match reliability (RCM) | ✅ 100% in S1/S3, 99.3% S2 [CNSFM] | ⚠️ Inconsistent | ⚠️ "Prone to mismatches" [CNSFM] | ⚠️ RCM as low as 52% [CNSFM] | ⚠️ Unknown |
| Crater-sparse robustness | ❌ Fails explicitly [CNSFM §P4] | ✅ Not crater-dependent | ✅ Not crater-dependent | ✅ Not crater-dependent | ✅ Not crater-dependent |
| Code availability | ✅ Public [github.com/Bin501/CNSFM] | ❌ Unknown | ❌ Unknown | ❌ Unknown | N/A |

**Why CNSFM over doing nothing:** LightGlue may handle most polar cases — but we don't know its lunar-fine-tuned ceiling. CNSFM provides a **known, quantified** fallback for the hardest cases. The cost of including it as a conditional path is low (activates only when needed), while the cost of NOT having it is potentially failing on the hardest test cases the PS is specifically designed to evaluate.

**Why not WSSF/HAPCG/ML-HLMO:** All substantially underperform CNSFM at the pole. HAPCG and ML-HLMO are described as "unstable and prone to mismatches" [CNSFM §P3]. WSSF is the closest competitor (31.2%) but still less than half CNSFM's success rate.

---

## Decision 3: Spatial Distribution Enforcement

**Decision:** ANMS-SSC (Suppression via Square Covering)

| Criterion | ANMS-SSC | Grid-based top-k | M-UR-SIFT uniform selection | Greedy NMS (LoFTR-style) | No enforcement |
|---|---|---|---|---|---|
| Coverage quality | ✅ Adaptive, confidence-weighted | ⚠️ Fixed grid — crude | ⚠️ Entropy-based — still SIFT-dependent [RIFT §P3] | ✅ Adaptive radius | ❌ Clusters in high-texture |
| Confidence awareness | ✅ Retains strongest per region | ❌ Arbitrary within cell | ✅ Entropy + contrast based | ✅ Score-based | N/A |
| Complexity | ✅ O(n), drop-in after any detector | ✅ Trivial | ⚠️ Requires adaptation from SIFT [DESCA §P1] | ⚠️ Conflict resolution unresolved [LoFTR_IMC21 §P1] | None |
| Code availability | ✅ `BAILOOL/ANMS-Codes` [Supplementary_Research §1] | ✅ ~20 lines of code | ❌ Separate cited paper | ⚠️ Partially described | N/A |
| Matcher-agnostic | ✅ Works on any keypoint set | ✅ Yes | ❌ Tied to SIFT pipeline | ⚠️ Designed for LoFTR output | N/A |

**Why ANMS-SSC over grid-based:** ANMS-SSC is confidence-weighted (keeps locally strongest match, not a random one), has O(n) runtime, and has ready-to-use code. Grid-based is the trivial fallback if ANMS has issues.

---

## Decision 4: Robust Geometric Estimator

**Decision:** DEGENSAC

| Criterion | DEGENSAC | Standard RANSAC | DESCA (DE optimization) | FSC (Fast Sample Consensus) | MAGSAC |
|---|---|---|---|---|---|
| Degeneracy handling | ✅ Purpose-built [LoFTR_IMC21 §P0] | ❌ Susceptible to dominant-plane | ❌ Not addressed [DESCA] | ❌ RANSAC-family [DESCA §P3] | ✅ Marginalizing sample consensus |
| Lunar relevance | ✅ Flat mare = dominant-plane risk | ⚠️ Risk on flat terrain | ⚠️ Fails on terrain relief [DESCA §P4] | ⚠️ Same as RANSAC | ⚠️ Untested on lunar |
| Sub-pixel RMSE | ⚠️ Depends on threshold (0.5 px) | ⚠️ Same | ✅ 0.67–0.95 px [DESCA §P0] | ⚠️ Inferior to DESCA [DESCA §P3] | ⚠️ Unknown for our case |
| Compute cost | ⚠️ Up to 10K iterations | ⚠️ Same | ❌ 200 DE generations per estimation [DESCA] | ✅ Faster than RANSAC | ⚠️ Moderate |
| Availability | ✅ pydegensac library, OpenCV USAC | ✅ OpenCV | ⚠️ Custom implementation needed | ⚠️ Less commonly available | ✅ OpenCV USAC |

**Why DEGENSAC over DESCA:** DESCA achieves better RMSE (0.67–0.95 px) but: (a) only validated on optical-optical same-modality [DESCA §P0], (b) only with affine model which fails on terrain relief [DESCA §P4], (c) requires careful two-tier initialization (random init causes divergence [DESCA §P2]), (d) 200 DE generations adds significant compute. DEGENSAC is simpler, handles the lunar-specific degeneracy risk, and is readily available. **DESCA is classified as an OPTIONAL upgrade** — investigate if DEGENSAC accuracy proves insufficient.

---

## Decision 5: Sub-Pixel Refinement

**Decision:** Phase-correlation with paraboloid fitting

| Criterion | Phase corr. + paraboloid | MI + DE optimization | LoFTR/LightGlue learned refinement | OpenCV cornerSubPix | Sinc interpolation |
|---|---|---|---|---|---|
| Demonstrated RMSE | ✅ < 0.2 px [HybridPhaseCorrelation §P0] | ✅ 0.57–0.82 px [KAZE §P0] | ⚠️ ~0.5–0.9 px (the matcher output) | ⚠️ Corner-specific, not general | ⚠️ Comparable to paraboloid [HybridPhaseCorrelation §P2] |
| Compute cost | ✅ ~1 ms per match (FFT of 64×64) | ❌ 200 DE generations (~30s total) [KAZE] | ✅ Already part of matching | ✅ Very fast | ✅ Fast |
| Training dependency | ✅ None | ✅ None | ⚠️ Requires trained model | ✅ None | ✅ None |
| Texture requirement | ⚠️ Needs some texture in patch | ⚠️ Needs MI signal | ✅ Works on learned features | ❌ Needs corners | ⚠️ Same as phase corr. |
| Evidence strength | ✅ Direct satellite imagery RMSE [HybridPhaseCorrelation] | ✅ Cross-sensor optical [KAZE] | ⚠️ Not validated as standalone refinement | ⚠️ Not validated for correspondence refinement | ⚠️ Tested but inferior to paraboloid [HybridPhaseCorrelation §P2] |

**Why phase correlation over MI+DE:** Phase correlation achieves better RMSE (<0.2 px vs 0.57–0.82 px) at dramatically lower compute (~1 ms vs ~30s). MI+DE is classified as OPTIONAL — investigate only if paraboloid fitting proves insufficient on specific lunar conditions.

---

## Decision 6: Front-End Detector (for LightGlue)

**Decision:** SuperPoint (primary) + Phase Congruency keypoints (supplementary, EXPERIMENTAL)

| Criterion | SuperPoint | PC (log-Gabor phase congruency) | SIFT | FAST | I-KAZE (PC-weighted) |
|---|---|---|---|---|---|
| LightGlue compatibility | ✅ Co-designed [SuperGlue §P1] | ⚠️ Compatible in principle — [PROPOSED INTEGRATION] | ✅ Supported [Additional_Resources §11] | ❌ Not a descriptor source | ⚠️ Custom adaptation needed |
| Illumination robustness | ⚠️ Trained on natural images — may miss lunar structures | ✅ Invariant by construction [RIFT §P0] | ❌ "Most sensitive to illumination" [CNSFM §P0] | ❌ 0 matches on 2/5 cross-sensor sets [KAZE §P4] | ✅ PC-weighted [KAZE §P0] |
| Low-texture robustness | ⚠️ Depends on training data distribution | ⚠️ PC fires on phase alignment, not gradient — somewhat better on subtle texture | ❌ Gradient-based — misses subtle lunar texture | ❌ Corner-only — worst on low texture | ⚠️ PC helps but KAZE itself is gradient-based |
| Pretrained availability | ✅ Widely available | ✅ No training needed | ✅ OpenCV | ✅ OpenCV | ⚠️ Custom implementation |

**Why supplementary PC keypoints are EXPERIMENTAL:** The combination has never been tested. It's architecturally sound because LightGlue's SuperPoint front end produces (keypoint, descriptor) pairs, and PC+MIM also produces (keypoint, descriptor) pairs that could be merged into the same input set. But unknown interactions could emerge — e.g., LightGlue's attention patterns may not adapt well to MIM descriptors without retraining. Validate in ablation experiment #3.

---

# 3. INTER-STAGE INTERFACE SPECIFICATIONS

These define the exact data contract between pipeline stages — what each stage produces and what the next stage expects.

```
┌─────────────────────────────────────────────────────┐
│ STAGE 0 → STAGE 1                                    │
│ Input contract:                                       │
│   source_image: PDS4 file path OR calibrated .cub    │
│   reference_dataset: {"WAC_mosaic" | "NAC_strip"}    │
│   metadata: {                                         │
│     corner_coords: [(lat,lon) × 4],                  │
│     solar_incidence_angle: float (degrees),          │
│     solar_azimuth_angle: float (degrees),            │
│     sensor_id: {"OHRC"|"TMC"|"IIRS"},                │
│     spatial_resolution_m: float                      │
│   }                                                   │
│ Output contract:                                      │
│   source_patch: np.ndarray (H×W, uint16 or float32) │
│   reference_patch: np.ndarray (H'×W', same dtype)    │
│   overlap_bbox: (lat_min,lat_max,lon_min,lon_max)    │
│   difficulty_flag: {"NORMAL"|"EXTREME"}              │
│   resolution_ratio: float                             │
│   solar_angle_diff: float (degrees)                  │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ STAGE 1 → STAGE 2                                    │
│ Input: source_patch, reference_patch, metadata       │
│ Output:                                               │
│   source_norm: np.ndarray (H×W, uint8)               │
│   reference_norm: np.ndarray (H×W, uint8)            │
│   # Both at matched resolution, normalized intensity │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ STAGE 2 → STAGE 3                                    │
│ Input: source_norm, reference_norm, difficulty_flag   │
│ Output:                                               │
│   correspondences: List[{                            │
│     x_src: float,                                    │
│     y_src: float,                                    │
│     x_ref: float,                                    │
│     y_ref: float,                                    │
│     confidence: float [0,1],                         │
│     source_method: {"LightGlue"|"CNSFM"}            │
│   }]                                                  │
│   match_count: int                                    │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ STAGE 3 → STAGE 4                                    │
│ Input: correspondences (unsorted, potentially        │
│        clustered)                                     │
│ Output:                                               │
│   distributed_correspondences: List[same schema]     │
│   # Spatially uniform subset                        │
│   coverage_metric: float (grid density std-dev)      │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ STAGE 4 → STAGE 5                                    │
│ Input: distributed_correspondences                    │
│ Output:                                               │
│   inlier_correspondences: List[same schema +         │
│     ransac_residual: float]                          │
│   transform_params: {                                │
│     model_type: {"similarity"|"affine"|"polynomial"},│
│     coefficients: np.ndarray,                        │
│     fit_rmse: float                                  │
│   }                                                   │
│   inlier_count: int                                   │
│   inlier_ratio: float                                │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ STAGE 5 → STAGE 6                                    │
│ Input: inlier_correspondences + original images      │
│ Output:                                               │
│   refined_correspondences: List[same schema +        │
│     subpixel_shift_x: float,                        │
│     subpixel_shift_y: float,                        │
│     correlation_peak: float,                         │
│     is_refined: bool  # False if rejected            │
│   ]                                                   │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ STAGE 6 → STAGE 7 (OUTPUT)                           │
│ Input: refined_correspondences                        │
│ Output:                                               │
│   final_correspondences: List[{                      │
│     x_src, y_src, x_ref, y_ref: float,              │
│     composite_confidence: float,                     │
│     is_subpixel_refined: bool                        │
│   }]                                                  │
│   transform: polynomial coefficients                  │
│   registered_image: np.ndarray (warped source)       │
│   quality_report: {                                  │
│     rmse: float,                                     │
│     medae: float,                                    │
│     pct_sub_1px: float,                              │
│     pct_sub_05px: float,                             │
│     inlier_count: int,                               │
│     inlier_ratio: float,                             │
│     spatial_coverage: float,                         │
│     success: bool                                    │
│   }                                                   │
└─────────────────────────────────────────────────────┘
```

---

# 4. SENSOR-PAIR CONFIGURATION PROFILES

Each sensor pair has fundamentally different characteristics. The pipeline parameters must adapt per pair.

## OHRC ↔ LRO NAC

| Parameter | Value |
|-----------|-------|
| Resolution ratio | 0.25m / 0.5–2m = **0.5–2×** (small) |
| Primary challenge | Illumination (different orbital passes, different Sun angles) |
| Modality gap | Small (both panchromatic optical) |
| Expected difficulty | MODERATE (equatorial), HIGH (polar) |
| Preprocessing | Percentile normalization. Resample NAC to OHRC resolution. |
| Recommended matcher | LightGlue (primary) |
| CNSFM activation | If solar azimuth diff > 60° |
| Expected RMSE target | < 1.0 px (pre-refinement), < 0.5 px (post-refinement) |
| Evidence | [Traditional_vs_DL]: SuperGlue RMSE 0.62/0.57 px on OHRC-NAC equatorial |

## TMC-2 ↔ LRO WAC

| Parameter | Value |
|-----------|-------|
| Resolution ratio | 5m / 75–100m = **15–20×** (large) |
| Primary challenge | Scale (large resolution gap) + illumination |
| Modality gap | Small-moderate (both panchromatic but different instruments) |
| Expected difficulty | MODERATE |
| Preprocessing | Resample WAC up to TMC resolution. Heavy percentile normalization. |
| Recommended matcher | LightGlue |
| CNSFM activation | If solar azimuth diff > 60° |
| Expected RMSE target | < 2.0 px in TMC pixels (< 10m ground) |
| Evidence | No direct TMC benchmark exists in any paper (confirmed gap across all research) |

## OHRC ↔ TMC-2

| Parameter | Value |
|-----------|-------|
| Resolution ratio | 0.25m / 5m = **20×** (large) |
| Primary challenge | Scale (20× resolution gap within same spacecraft) |
| Modality gap | Negligible (both Chandrayaan-2 panchromatic optical) |
| Expected difficulty | MODERATE (scale-dominated) |
| Preprocessing | Resample TMC-2 up 20×. Bilinear for low Sun; bicubic for high Sun [MoonMetaSync §P2]. |
| Recommended matcher | LightGlue at matched resolution |
| CNSFM activation | Usually not needed (same-spacecraft, similar illumination) |
| Expected RMSE target | < 1.0 px in OHRC pixels (< 0.25m ground) |
| Evidence | [MoonMetaSync] demonstrated OHRC↔TMC-2 geodesic pre-matching + affine alignment. SIFT > IntFeat > ORB. |

## IIRS ↔ LRO WAC

| Parameter | Value |
|-----------|-------|
| Resolution ratio | 80m / 100m = **1.25×** (small) |
| Primary challenge | Modality (hyperspectral vs panchromatic) + illumination |
| Modality gap | LARGE (250 spectral bands vs single panchromatic) |
| Expected difficulty | HIGH (cross-spectral matching) |
| Preprocessing | Band selection (choose most texture-rich IIRS band, typically visible-range). Percentile normalization. Statistic transfer [SIFT-IIRS-WAC §P0]. |
| Recommended matcher | LightGlue with selected IIRS band |
| CNSFM activation | IIRS resolution (80m) may be too coarse for crater detection — skip CNSFM |
| Expected RMSE target | < 1.0 px in IIRS pixels (< 80m ground) — [SIFT-IIRS-WAC] achieved ~73m |
| Evidence | [SIFT-IIRS-WAC] demonstrated SIFT+RANSAC achieving mean RMSE ~73m (sub-pixel) across ~200 strips |

## OHRC ↔ IIRS (EXTREME — handle separately)

| Parameter | Value |
|-----------|-------|
| Resolution ratio | 0.25m / 80m = **320×** (extreme) |
| Primary challenge | Extreme scale gap + modality gap |
| Modality gap | EXTREME |
| Expected difficulty | VERY HIGH |
| Preprocessing | **Transitive registration:** OHRC→NAC→WAC→IIRS (chain transformations via intermediate references) |
| Recommended matcher | NOT direct matching. Use transitive path: match OHRC↔NAC, match IIRS↔WAC, chain via NAC↔WAC (which is pre-registered in LRO reference frame). |
| Expected RMSE target | Accumulated error from chain — likely ~2–5 px in IIRS pixels |
| Evidence | No paper tests this extreme pair. [NASA_SubPixel §P2]: "registration accuracy degrades significantly when footprint ratio exceeds ~2–3×." [KAZE §P4]: I-KAZE+MI/DE failed on optical-vs-SAR (large modality gap analogy). |

---

# 5. RISK REGISTER

| # | Risk | Likelihood | Impact | L×I Score | Mitigation | Contingency |
|---|------|-----------|--------|-----------|------------|-------------|
| R1 | LightGlue (trained on MegaDepth natural photos) fails to generalize to lunar terrain without fine-tuning | HIGH | HIGH | **9** | Fine-tune on synthetic lunar data (see §6). Test early on OHRC↔NAC pairs. | Fall back to RIFT2 + CNSFM (Architecture B). |
| R2 | SuperPoint detects too few repeatable keypoints on low-texture lunar mare | HIGH | MEDIUM | **6** | Supplement with PC keypoints (Stage 3A). Use grid-based detection as last resort. | Switch to DISK or ALIKED front-end (both supported by LightGlue). |
| R3 | CNSFM crater detector fails on OHRC/TMC (trained only on LROC NAC) | MEDIUM | MEDIUM | **4** | Fine-tune YOLOv9 on OHRC data. Use DeepMoon [Additional_Resources §12] or TMC-2 crater U-Net [Additional_Resources §13] as alternatives. | Drop CNSFM; accept reduced polar success rate. |
| R4 | No ground-truth correspondence data exists for evaluation | HIGH | HIGH | **9** | Generate synthetic transforms (Phase 1). Use held-out checkpoints (Phase 2). Cross-reference against ASP results (Phase 3). | Synthetic-only evaluation — valid but less convincing. |
| R5 | OHRC/TMC/IIRS raw data acquisition from ISRO is delayed or difficult | MEDIUM | HIGH | **6** | Use PRADAN servers [MoonMetaSync] + ISSDC. Start with publicly available LRO NAC/WAC data for pipeline development. | Develop/test entirely on LRO NAC↔WAC pairs (same pipeline, different input sensors). |
| R6 | Phase-correlation sub-pixel refinement fails on low-texture lunar patches | MEDIUM | MEDIUM | **4** | Reject unreliable refinements (correlation < threshold). Retain coarse match position. | Switch to MI+DE refinement [KAZE] for difficult patches — slower but statistically robust. |
| R7 | Extreme resolution ratio (OHRC↔IIRS, 320×) defeats all direct matching | HIGH | LOW | **3** | Transitive registration via intermediate references (OHRC→NAC→WAC→IIRS). | Treat IIRS as separate module with lower accuracy target. |
| R8 | DEGENSAC/RANSAC threshold requires per-sensor-pair tuning | MEDIUM | LOW | **2** | Start with 0.5 px threshold [HybridPhaseCorrelation]. Tune based on initial results per sensor pair. | Use conservative threshold (1.0 px) and accept slightly lower outlier rejection. |
| R9 | GPU unavailable for LightGlue inference | LOW | HIGH | **3** | LightGlue is designed for efficiency (~17ms on GPU). CPU inference is possible but ~10× slower. | Architecture B (RIFT2 + I-KAZE) is entirely CPU-based. |
| R10 | Lunar curvature causes near-exponential error growth beyond ±55° latitude [SIFT-IIRS-WAC §P2] | HIGH (for polar targets) | MEDIUM | **6** | Use piecewise polynomial (per-tile) rather than global transform. | Acknowledge as documented limitation. Report accuracy by latitude band. |
| R11 | ANMS over-suppresses, leaving too few matches for DEGENSAC | LOW | MEDIUM | **2** | Set minimum match count floor (30). Widen suppression radius if below threshold. | Fall back to grid-based top-k (always produces target count). |
| R12 | Pipeline integration complexity — 7 stages with conditional paths | MEDIUM | MEDIUM | **4** | Modular implementation. Each stage independently testable. Build incrementally (Phase 1→6 roadmap). | Simplify: drop CNSFM conditional path, drop PC supplements. Core pipeline is 5 stages. |

---

# 6. TRAINING DATA STRATEGY FOR LIGHTGLUE FINE-TUNING

LightGlue's pretrained weights come from MegaDepth (natural outdoor photos). For optimal lunar performance, fine-tuning is recommended [Traditional_vs_DL: "requires domain-specific training for optimal results"].

## Data Sources (ranked by priority)

### Source 1: Synthetic Lunar Image Pairs with Controlled Transforms (HIGHEST PRIORITY)

**Method:** Take a single real lunar image (OHRC or LRO NAC), apply known geometric transforms (rotation, scale, translation, affine) + photometric transforms (brightness, contrast, gamma to simulate Sun-angle variation). The applied transform is the ground truth.

**Advantages:**
- Unlimited quantity. Exact ground truth. No manual annotation.
- Can control difficulty level (small vs large transforms).
- [Additional_Resources §4] identifies open-source tools for synthetic lunar rendering with controllable Sun angle.

**Limitations:**
- Synthetic photometric variation may not capture the full complexity of real Sun-angle-driven shadow patterns.
- No real cross-sensor modality gap.

**Use:** Phase 1 pipeline validation + initial LightGlue fine-tuning.

### Source 2: ASP-Derived Correspondence Ground Truth

**Method:** Run NASA ASP's stereo pipeline on known OHRC stereo pairs [Additional_Resources §1-2]. ASP's bundle adjustment produces independently verified correspondence points — these become ground-truth training pairs.

**Advantages:**
- Real data. Real sensor. Geometrically validated correspondence.
- [Additional_Resources §1] achieves ~30m horizontal accuracy on OHRC.
- ASP's `--ip-per-tile` feature produces spatially distributed interest points — directly usable as ground-truth correspondence.

**Limitations:**
- Requires ASP installation + OHRC stereo pairs.
- Limited to OHRC sensor (not cross-sensor).
- ASP correspondence accuracy is the ceiling (can't train beyond ASP's own quality).

**Use:** Phase 2 fine-tuning on real OHRC data.

### Source 3: Cross-Sensor Bootstrapping

**Method:** Use the pipeline itself (Architecture C baseline or early LightGlue) to match OHRC↔NAC equatorial pairs (the easiest case). Manually verify a subset. Use verified matches as training pairs for harder conditions.

**Advantages:**
- Generates real cross-sensor correspondences.
- Can iteratively improve: easy pairs → train → harder pairs → retrain.

**Limitations:**
- Circular if not carefully managed (training on your own output).
- Manual verification is labor-intensive.

**Use:** Phase 3 cross-sensor fine-tuning.

### Source 4: Physically-Based Rendering (FUTURE)

**Method:** Use DEMs (from LRO NAC stereo or ASP-processed OHRC) + a lunar photometric model (Hapke reflectance) to render synthetic images at arbitrary Sun angles. The rendering parameters are the ground truth.

**Advantages:**
- Physically realistic illumination variation.
- Can generate arbitrary Sun-angle pairs for the same terrain.
- [Additional_Resources §4] evaluates open-source tools for exactly this.
- [Radiometric_Normalization §P2] mentions "physics-informed learning strategies incorporating photometric models of lunar illumination" as unexplored — this would be our contribution.

**Limitations:**
- Complex implementation. Requires accurate DEMs.
- Rendering fidelity vs reality gap.

**Use:** Phase 3–4 for extreme illumination robustness training.

---

## Fine-Tuning Strategy

1. **Phase 1:** Train on synthetic transforms of real OHRC/NAC images (Source 1). This gives LightGlue basic familiarity with lunar texture patterns without requiring real paired data.

2. **Phase 2:** Fine-tune on ASP-derived real OHRC correspondences (Source 2). This introduces real-world noise, distortion, and sensor artifacts.

3. **Phase 3:** Add cross-sensor bootstrapped pairs (Source 3) for OHRC↔NAC modality awareness.

4. **Phase 4 (if time permits):** Add physically-rendered multi-illumination pairs (Source 4) for Sun-angle robustness.

**Expected training data volume:**
- Synthetic: 10K+ pairs (trivial to generate)
- ASP-derived: 500–1K pairs (limited by ASP processing time)
- Bootstrapped: 200–500 verified pairs (limited by manual verification)
- Rendered: 1K+ pairs (if implemented)

**Training infrastructure:**
- Single GPU (RTX 3090 or equivalent) sufficient.
- Fine-tuning (not training from scratch): ~4–8 hours per stage.
- LightGlue supports fine-tuning via the `hloc` + PyTorch ecosystem.

---

*This companion document provides the technical depth needed to move from architecture to implementation. Every classification, decision, interface, and risk mitigation is traceable to the 13 research extractions analyzed.*
