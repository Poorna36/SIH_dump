# MoonMetaSync (extracted)

## 1. RELEVANCE SNAPSHOT
- Overall relevance: High
- Most relevant component: Comparative benchmark of SIFT vs. ORB vs. custom hybrid (IntFeat) for OHRC↔TMC-2 (Chandrayaan-2) cross-resolution registration, plus a documented failure mode: low Sun-angle images degrade both feature detection and interpolation.
- Why it matters: Directly tests the exact sensor pair (OHRC 30 cm vs. TMC-2 5 m) and exact challenge (scale + illumination) named in our problem statement, and gives a negative result for a "combine SIFT+ORB" hybrid — useful P4 evidence, not just a P0 win.
- Most valuable takeaway: [DEMONSTRATED] Plain SIFT outperformed both ORB and the authors' novel hybrid detector (IntFeat) on lunar imagery in nearly all conditions, and all methods degrade under low solar-angle (high-contrast/shadow) images — reinforcing Paper 1's illumination-mismatch finding from an independent dataset/method.

## 2. P0 — DIRECT CONNECTION / USE NOW

### OHRC↔TMC-2 Geodesic Pre-Matching + Affine Alignment
- WHAT: Corner-coordinate-based geographic pre-matching of OHRC (high-res) patches within TMC-2 (low-res) frames using `pygeodesy`, followed by geodesic interpolation of pixel coordinates (every 100th pixel) and Haversine-distance-based spatial matching, then affine transform to align OHRC into TMC-2 frames.
- WHY IT CONNECTS: This is a direct, demonstrated method for our exact sensor pair (OHRC/TMC-2) and addresses scale variation via coordinate-based coarse alignment before feature matching — a practical two-stage (coarse geodesic + fine feature-based) registration strategy.
- WHAT WE COULD USE: The coarse geodetic pre-alignment step as a bounding-box/ROI-narrowing stage before feature detection, reducing search space and false-match risk — conceptually complements Paper 1's tiling approach.
- Evidence: [REPORTED] Produced 16 OHRC images matched to 6 TMC-2 images with defined bounding boxes; used as preprocessing before feature detection, not evaluated in isolation for geometric accuracy.
- Implementation consideration: Relies on availability of accurate corner-coordinate metadata for both sensors — reintroduces the same metadata-reliability risk flagged as a failure mode in Paper 1.

### SIFT vs. ORB vs. IntFeat Benchmark on Lunar Patches (Bi-linear/Bi-cubic Upscaling)
- WHAT: Quantitative comparison (SSIM, PSNR) of three feature detectors after upscaling low-res TMC-2 patches (128×128) to match OHRC high-res patches (1024×1024) via bilinear/bicubic interpolation, tested on both low-angle and high-angle (Sun) image sets.
- WHY IT CONNECTS: Directly benchmarks feature-detector choice AND upscaling method under scale mismatch — both explicit SIH requirements (scale invariance, multi-resolution sensors).
- WHAT WE COULD USE: The benchmark result itself as a decision input: SIFT > IntFeat > ORB across nearly all conditions tested (see P9 table); also the recommendation that bilinear upscaling be paired with SIFT/IntFeat while bicubic favors high-angle/high-detail images.
- Evidence: [DEMONSTRATED] Table I (low-angle): SIFT SSIM 0.7694/0.7585 (bilinear/bicubic) vs. ORB 0.7554/0.7475 vs. IntFeat 0.7633/0.7380. Table II (high-angle): SIFT 0.7887/0.7825 vs. ORB 0.7750/0.7653 vs. IntFeat 0.7880/0.7812.
- Implementation consideration: Metrics used (SSIM/PSNR) evaluate registered-image visual/structural similarity, not geometric correspondence accuracy (RMSE/inlier ratio as required by our SIH deliverable) — a methodological gap to note (see P4).

## 3. P1 — HIGHLY RELEVANT / INTEGRABLE

### Sun-Angle-Based Image Filtering as a Pre-Processing Gate
- WHAT: Filtering input image pairs by solar elevation angle (using satellite metadata) to exclude low-angle (high-contrast/deep-shadow) images before registration.
- WHY RELEVANT: Provides an operationally simple, dataset-level mitigation for the Sun-angle problem central to our project, complementary to (not a replacement for) algorithmic illumination robustness.
- HOW IT COULD INTEGRATE: Could serve as a data-selection/quality-gating step in our pipeline — flag or deprioritize image pairs with large predicted incidence-angle mismatch or very low absolute Sun elevation, before attempting feature matching.
- WHAT WOULD NEED TO CHANGE: Not always feasible — our problem requires robustness across arbitrary Sun-angle differences, not just curation of "easy" cases; this is a workaround for benchmarking, not a general solution.
- ADVANTAGE: Simple, no algorithmic change needed, immediately improves measured performance (Table I → Table II improvement).
- CHALLENGE: Reduces usable data coverage; doesn't solve the underlying illumination-invariance problem, only avoids the hardest cases.
- OPPORTUNITY: Use as a documented limitation/operating envelope in our own solution (e.g., report accuracy conditional on Sun-angle bins), rather than a fix.

## 4. P2 — RESEARCH THEORY / TECHNICAL INSIGHT

### Interpolation Method Interacts With Illumination Conditions
- CONCEPT: Bicubic interpolation (theoretically superior, larger 4×4 neighborhood) does not universally outperform bilinear on lunar imagery — its advantage is conditional on Sun angle/contrast.
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED] On low-angle (high-contrast/shadow) images, bicubic performed worse than bilinear; on high-angle images, bicubic outperformed bilinear as expected (finer detail preservation).
- WHY IT MATTERS: General image-processing assumptions (bicubic > bilinear) don't transfer cleanly to lunar surface imagery — sharp shadow boundaries cause bicubic's wider neighborhood to amplify contrast artifacts during upscaling.
- PRACTICAL IMPLICATION: [INFERENCE] If our pipeline needs to upscale lower-resolution sensor data (e.g., IIRS/TMC) to match higher-resolution references (e.g., OHRC), interpolation method should be selected conditionally on illumination geometry, not fixed a priori.

### Noise Amplification in Hybrid/Low-Level Feature Detectors
- CONCEPT: Combining low-level (ORB/BRIEF-style) and high-level (SIFT) features doesn't guarantee additive benefit on noisy, high-contrast domains.
- WHAT THE PAPER ESTABLISHES: [REPORTED] IntFeat's ORB-derived low-level features amplify noise/artifacts, particularly worsened during interpolation-based smoothing, which the authors identify as the mechanism behind IntFeat's underperformance vs. SIFT.
- WHY IT MATTERS: Cautions against naively combining descriptor types as a robustness strategy for lunar imagery without explicit noise-handling.
- PRACTICAL IMPLICATION: [INFERENCE] If we explore hybrid/ensemble feature detectors (as considered in Paper 1's P5 section), noise-robustness must be explicitly engineered, not assumed to emerge from combining complementary detectors.

## 5. P3 — EXISTING / PAST SOLUTIONS

### SIFT (Baseline, ref. Lowe 2004)
- APPROACH: Classical scale/rotation-invariant keypoint detector + descriptor, four-stage pipeline (scale-space extrema, localization, orientation assignment, descriptor generation).
- RESULT: [DEMONSTRATED] Best performer of the three methods tested in nearly every condition (both interpolation types, both angle conditions).
- STRENGTH: Highest SSIM/PSNR scores; robust to scale and rotation as expected.
- LIMITATION: Computationally intensive (acknowledged but not directly benchmarked for speed in this paper).
- RELEVANCE TO US: Reinforces Paper 1's choice of SIFT as a sound baseline; independent confirmation on a different sensor pair (OHRC/TMC-2 vs. Paper 1's IIRS/WAC).

### ORB (Baseline, ref. Rublee et al. 2011)
- APPROACH: FAST keypoint detector + BRIEF binary descriptor, with rotation invariance added; designed for computational efficiency over SIFT.
- RESULT: [DEMONSTRATED] Consistently lowest SSIM/PSNR of the three methods across all conditions tested.
- STRENGTH: Computationally lighter (real-time capable), though speed itself wasn't benchmarked here.
- LIMITATION: [DEMONSTRATED] Sacrifices feature representation richness, weakest performance in this lunar domain.
- RELEVANCE TO US: Suggests ORB alone is a weaker choice for our accuracy-critical (sub-pixel) SIH deliverable unless speed is a hard constraint; potentially usable in a fast coarse-alignment stage before a more accurate refinement step.

## 6. P4 — WHAT NOT TO TRY / FAILURE MODES

### IntFeat: Naive SIFT+ORB Descriptor Fusion (PCA-reduced SIFT + ORB, combined via brute-force matching)
- METHOD: Extract SIFT and ORB keypoints/descriptors separately, PCA-reduce SIFT descriptors to match ORB dimensionality, combine into a joint vector space, then brute-force match + RANSAC homography.
- WHY PEOPLE USE IT: Intended to get "best of both worlds" — SIFT's rich features + ORB's efficiency.
- FAILURE / WEAKNESS: [DEMONSTRATED] Did not exceed SIFT's accuracy in any tested condition; underperformed SIFT in both low-angle and high-angle scenarios (though it did beat ORB).
- WHY IT FAILS: [REPORTED] Struggles with the combination of high noise, extreme contrast, homogeneous textures, and fluctuating lighting simultaneously; ORB-derived low-level features amplify noise, especially exacerbated by interpolation-based smoothing; PCA dimensionality reduction of SIFT descriptors likely also discards discriminative information.
- CONDITIONS WHERE IT FAILS: Lunar surface imagery generally — high-contrast regions, low-angle illumination, noisy/homogeneous terrain textures.
- IMPLICATION FOR OUR PROJECT: [INFERENCE] Simple feature-fusion (concatenation/PCA-matching of heterogeneous descriptor types) is not a reliable path to improved robustness for lunar imagery; more sophisticated fusion (e.g., learned joint embeddings, late-stage match fusion/voting rather than early descriptor concatenation) would likely be needed if a hybrid approach is still desired.
- POSSIBLE ALTERNATIVE: Use SIFT as primary detector; consider ORB only as a fast pre-filter/coarse stage, not a co-equal fused descriptor.

### SSIM/PSNR as Primary Registration-Quality Metrics
- METHOD: Evaluating registration quality via image-similarity metrics (SSIM, PSNR) between registered and reference images.
- WHY PEOPLE USE IT: Standard, easy-to-compute image quality metrics; convenient when ground-truth correspondence points aren't available.
- FAILURE / WEAKNESS: [INFERENCE] SSIM/PSNR measure overall pixel/structural similarity post-warp, not geometric correspondence accuracy (sub-pixel offset, inlier ratio) — they can be high even with imperfect local alignment, or penalized by legitimate radiometric differences unrelated to misregistration.
- WHY IT FAILS (for our purposes): Our SIH deliverable explicitly requires RMSE, inlier count, and inlier ratio as evaluation metrics — SSIM/PSNR don't directly quantify sub-pixel correspondence accuracy or spatial match distribution.
- CONDITIONS WHERE IT FAILS: Whenever precise quantitative geometric accuracy (not just visual similarity) is the deliverable — exactly our case.
- IMPLICATION FOR OUR PROJECT: Do not adopt SSIM/PSNR as our primary/sole accuracy metric; use them (if at all) as secondary/qualitative indicators alongside RMSE/inlier-ratio metrics from Paper 1's approach.
- POSSIBLE ALTERNATIVE: RMSE + inlier ratio + inlier count via GCP/ICP validation, as demonstrated in Paper 1.

## 9. SOLUTION / DECISION MATRIX

| Finding / Method | What it solves | Why useful | Why NOT use it | Challenge | Opportunity | Relevance |
|---|---|---|---|---|---|---|
| Geodetic coarse pre-alignment (pygeodesy + Haversine) | Scale/search-space reduction for OHRC↔TMC-2 | Directly demonstrated on our exact sensor pair | Depends on accurate corner metadata (same risk as Paper 1) | Metadata reliability across sensors | Combine as Stage 1 before tiled-SIFT fine registration | P0/P5 |
| SIFT vs ORB vs IntFeat benchmark | Detector selection guidance | Confirms SIFT as strongest baseline on lunar imagery, independent of Paper 1 | N/A — supports our existing direction | None major | Use as corroborating evidence for SIFT-first strategy | P0/P3 |
| IntFeat (naive descriptor fusion) | Attempted robustness via hybridization | Cautionary case study | Underperforms plain SIFT; amplifies noise | Noise/contrast/texture combination in lunar imagery | Redesign fusion at match/decision level instead of descriptor level | P4 |
| SSIM/PSNR as registration metric | Convenient similarity scoring | Easy to compute, no GCPs needed | Doesn't measure geometric/sub-pixel correspondence accuracy required by SIH | — | Use only as secondary qualitative check, not primary metric | P4 |
| Sun-angle-conditional interpolation choice | Upscaling quality under variable illumination | Demonstrated bicubic/bilinear trade-off is angle-dependent | Untested beyond 2 coarse bins | Needs threshold calibration | Build adaptive interpolation stage keyed to solar geometry metadata | P5 |

## 10. KEY TAKEAWAYS FOR OUR PROJECT
1. [USE NOW] SIFT again outperforms alternatives (here, ORB and a custom hybrid) on lunar imagery — second independent confirmation (different dataset/sensors than Paper 1) that SIFT is a sound default baseline detector.
2. [USE NOW] Geodetic/coordinate-based coarse pre-alignment (bounding box + Haversine matching from corner metadata) is a viable, demonstrated Stage-1 step for OHRC↔TMC-2 specifically — worth combining with Paper 1's fine-registration pipeline.
3. [AVOID] Naive early-fusion of SIFT+ORB descriptors (IntFeat's approach: PCA-match dimensions + concatenate) does not improve on plain SIFT and amplifies noise — do not pursue this specific fusion strategy without redesign.
4. [AVOID] SSIM/PSNR alone as the primary accuracy metric — doesn't satisfy our SIH requirement for RMSE/inlier-ratio-based quantitative correspondence accuracy.
5. [CONFIRMS/COMPLEMENTS PAPER 1] Independent confirmation that Sun-angle/illumination geometry is a first-order performance driver — here shown to also affect upscaling/interpolation quality, not just feature matching, broadening the scope of "illumination robustness" our solution needs to address (both detection AND any resampling steps).
6. [RESEARCH GAP] This paper filters out hard (low Sun-angle) cases to show improvement — our project cannot do this and must handle the full illumination range; no tested solution here generalizes across the full range without curation.
7. [RESEARCH GAP] Neither this paper nor Paper 1 evaluates a sensor pair with truly extreme scale ratio and multi-sensor combination simultaneously (this paper: ~17x scale ratio OHRC:TMC-2, single modality pair; Paper 1: ~1.25x scale ratio, but does cross IIRS-hyperspectral↔WAC). Our project may need to test the union of large scale gaps + spectral/modal difference not fully covered by either.
8. [METHODOLOGY NOTE] Small sample sizes in both papers (Paper 1: 8 detailed strips of ~200; this paper: 16 OHRC/6 TMC-2 patches) suggest we should plan a larger, more systematic validation set for our own benchmarking to draw robust conclusions.
