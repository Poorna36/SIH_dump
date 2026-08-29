# A Hybrid Multi-Scale Phase-Correlation Framework for Sub-Pixel Registration of Multi-Temporal VHR Remote Sensing Images (Rasmy, Sebari & Ettarid, 2026)

## 1. RELEVANCE SNAPSHOT
- Overall relevance: Very High
- Most relevant component: A complete, working, non-learned hybrid registration pipeline (Fusion multi-detector + coarse-to-fine multi-scale phase correlation + per-level and final sub-pixel refinement + RANSAC) evaluated with the exact same metric family named in the SIH brief (RMSE, inlier ratio, sub-pixel accuracy %, MedAE), on real optical satellite imagery with genuine terrain-induced geometric distortion and differing solar illumination angles between acquisitions.
- Why it matters to OUR project: This is the closest analog yet to our actual target scenario — real remote-sensing image pairs (not natural photos), evaluated with the metrics the SIH brief explicitly names, including a direct head-to-head comparison against LoFTR (Paper 2's method) and SIFT, plus an explicit study of how terrain relief degrades correspondence accuracy — directly analogous to lunar terrain/topography effects.
- Most valuable takeaway: A well-tuned classical/analytical (non-learned) pipeline can match or outperform a state-of-the-art transformer-based detector-free matcher (LoFTR) on VHR satellite sub-pixel registration, while avoiding LoFTR's observed failure mode of producing out-of-image-domain (extrapolated, unstable) correspondences — suggesting classical/hybrid methods remain a strong, resource-light candidate for our project, especially where large annotated lunar training sets are unavailable.

## 2. P0 — DIRECT CONNECTION / USE NOW

### Full hybrid pipeline: Fusion detector → multi-scale coarse-to-fine phase correlation → sub-pixel refinement → RANSAC
- WHAT: End-to-end registration pipeline: (1) preprocessing (grayscale, histogram equalization, Gaussian blur, CLAHE, high-pass filtering, patch normalization); (2) a new "Fusion" corner detector combining FAST + Shi–Tomasi keypoints, filtered by Canny edge detection + probabilistic Hough line transform; (3) phase-only correlation matching across a 3-level Gaussian pyramid, refined per level with 1D parabolic fitting and accumulated across scales; (4) final 2D paraboloid sub-pixel fitting on a 3×3 window around the correlation peak; (5) duplicate-match removal (one-to-one constraint); (6) global RANSAC (similarity transform, 0.5 px threshold, 5000 iterations) for outlier rejection; (7) polynomial transformation model fit by least squares.
- HOW IT WORKS: Phase-only correlation (normalized cross-power spectrum) is computed in the frequency domain per patch/scale; the peak location gives integer/coarse displacement; sub-pixel corrections are layered on top (parabolic → paraboloid) exactly as in the coarse-to-fine philosophy repeatedly confirmed across all prior papers.
- WHY IT CONNECTS TO OUR REQUIREMENTS: This single pipeline directly implements every major SIH requirement simultaneously: sub-pixel accuracy, outlier rejection (RANSAC), a detector aimed at feature repeatability/robustness (Fusion), a documented uniform-ish spatial candidate distribution via cell-based spatial indexing, and quantitative evaluation via RMSE/inlier ratio/MedAE/sub-pixel %.
- WHAT WE COULD USE: The overall architecture as a candidate reference pipeline for a classical/hybrid (non fully-learned) baseline system for OHRC/TMC/IIRS correspondence, particularly valuable given lunar training data scarcity relative to Earth remote sensing.
- Evidence from paper: [DEMONSTRATED] RMSE below 0.2 px (Sentinel-2) and 0.4 px (Pleiades) against real GCP-based validation; outperforms standalone SIFT and is competitive with/more balanced than LoFTR (Table 5).
- Potential implementation consideration: Designed for same-sensor, same-modality multi-temporal pairs (Pleiades↔Pleiades, Sentinel↔Sentinel) — not yet demonstrated cross-sensor/multi-modal, same gap flagged in Papers 2–3.

### Fusion multi-detector (FAST + Shi–Tomasi + Canny edges + probabilistic Hough line filtering)
- WHAT: Combines FAST (dense, fast) and Shi–Tomasi (stable, repeatable) keypoints at multiple scales; retains only Shi–Tomasi points within 3 px of a Hough-detected line segment; merges with FAST keypoints; deduplicates.
- WHY IT CONNECTS: Directly addresses the SIH requirement for reliable, well-distributed feature detection; the ablation study provides quantified evidence that combining a dense detector with a stability-filtering step (edges/lines) substantially improves downstream registration quality, not just raw match count.
- WHAT WE COULD USE: The general design pattern — dense candidate generation + geometric-structure-based filtering — as a detector strategy for lunar images, where craters/ridges/scarps could substitute for the Hough-line filtering criterion (e.g., filtering by proximity to crater rims or ridge lines instead of straight lines).
- Evidence from paper: [DEMONSTRATED, ablation Table 3] Full Fusion detector achieves RMSE = 0.010 px and inlier ratio 0.406 — an order of magnitude better RMSE than Shi–Tomasi alone (0.309 px) or FAST alone (0.258 px) or their simple combination (0.266 px), showing the edge/line filtering step (not just combining detectors) is what drives the largest gain.
- Potential implementation consideration: The probabilistic-Hough-line filter is tuned for man-made/urban structures (roads, buildings) in these VHR datasets; would need redesign for natural lunar terrain features (craters, ridges) rather than straight lines.

### RANSAC-based outlier rejection with tuned similarity-transform threshold
- WHAT: Global RANSAC over merged tile correspondences, estimating a similarity transform with a 0.5-pixel reprojection error threshold and up to 5000 iterations.
- WHY IT CONNECTS: Directly satisfies the SIH "outlier rejection" requirement with concrete, reproducible parameter choices validated on VHR imagery.
- WHAT WE COULD USE: The specific threshold/iteration choices as a starting point for our own RANSAC tuning, and the "compute-first-then-globally-verify" tile-merging strategy (RANSAC applied only after merging all tile-level correspondences, for global consistency).
- Evidence from paper: [REPORTED] "Most inliers showing registration errors below half a pixel" after RANSAC.
- Potential implementation consideration: Standard RANSAC (not a degeneracy-aware variant like DEGENSAC from Paper 2) — worth combining with DEGENSAC-style safeguards given lunar mare/flat-terrain risk of dominant-plane degeneracy.

### Evaluation metric suite matching the SIH brief almost exactly
- WHAT: RMSE, sub-pixel accuracy percentage (% of points under 1 px and under 0.5 px), Median Absolute Error (MedAE), Inlier Ratio (Ninliers/Nmatches), and an "Improvement" metric (% RMSE reduction from sub-pixel refinement).
- WHY IT CONNECTS: This is the most direct match yet to SIH 26166's explicitly named metrics (RMSE, inlier count/ratio) — a ready-made, validated evaluation protocol template.
- WHAT WE COULD USE: Adopt this exact metric set (RMSE, %<1px, %<0.5px, MedAE, inlier ratio, and an explicit "improvement" metric showing the value added by sub-pixel refinement specifically) for our own evaluation report.
- Evidence from paper: [DEMONSTRATED] Table 1 defines each metric with formulas; used consistently throughout all experiments (Tables 3–5).
- Potential implementation consideration: Requires reliable ground-truth/reference points (here: manually selected GCPs on stable, well-defined features) — for lunar imagery, equivalent GCPs would need to come from co-registered reference data (cf. Paper 1's laser-DTM co-registration, or existing LRO NAC-based control points).

## 3. P1 — HIGHLY RELEVANT / POTENTIALLY INTEGRABLE

### Terrain-relief-stratified accuracy analysis (gentle / hilly / mountainous slope classes)
- WHAT: Registration accuracy explicitly measured across three slope classes (5–15%, 15–30%, 30–50%), showing RMSE increases from (0.040, 0.050) px in gentle terrain to (0.179, 0.143) px in mountainous terrain, with inlier ratio dropping from 2.68 to 0.47.
- WHY RELEVANT: Directly analogous to the lunar terrain-relief problem — cratered highlands vs. flat mare are geometrically similar to mountainous vs. gentle terrain classes here.
- HOW IT COULD CONNECT/INTEGRATE: We could design an analogous stratified evaluation for lunar test regions (e.g., mare vs. highland vs. crater-rim terrain) to characterize where our correspondence method is reliable vs. where it hits geometry-driven limits.
- WHAT WOULD NEED TO CHANGE: Terrain classification criteria would need lunar-specific slope/roughness data (e.g., from existing DTMs, cf. Paper 1).
- ADVANTAGE: Provides an honest, terrain-stratified accuracy characterization rather than a single global number — more scientifically defensible reporting for our SIH deliverable.
- CHALLENGE: Requires DTM/slope data for the lunar test regions used in evaluation.
- OPPORTUNITY: [REPORTED] The authors explicitly conclude that in severe terrain + large viewing-angle differences, "the registration error is primarily governed by the imaging geometry rather than by the registration algorithm itself" and that further gains would require DEMs or advanced geometric correction models — directly connects back to Paper 1's laser-DTM co-registration as a potential complementary technique for extreme-terrain lunar regions.

### Apodization/windowing function study for phase correlation
- WHAT: Systematic comparison of Rectangular (no window), Hanning, Hamming, Blackman, Gaussian, and Tukey windows applied before phase correlation.
- WHY RELEVANT: Direct, quantified guidance on a specific implementation choice (spectral windowing) relevant if we adopt any phase-correlation-based component in our pipeline.
- HOW IT COULD CONNECT/INTEGRATE: If phase correlation is used (e.g., as a sub-pixel refinement step, echoing Paper 1's height-based sub-pixel refinement, now shown in a native 2D image context), Gaussian/Tukey windowing is the evidence-backed default.
- WHAT WOULD NEED TO CHANGE: Re-validation on lunar image statistics (different noise/texture characteristics than VHR Earth imagery).
- ADVANTAGE: Concrete, ready-to-use guidance (avoid Blackman; prefer Gaussian or Tukey).
- CHALLENGE: Trade-off is imagery-dependent; the paper itself notes performance "affects performance in complex ways," so this needs re-validation, not blind transfer.
- OPPORTUNITY: [DEMONSTRATED] Gaussian apodization gives the best overall RMSE/sub-pixel-ratio balance (1.163 px RMSE, 49.54% sub-pixel with SIFT); Blackman performs worst because it "suppresses too much high-frequency information" — directly useful if high-frequency lunar surface texture (e.g., small craters, boulders) is similarly important to preserve.

### Pyramid-type comparison (Gaussian vs. Laplacian vs. Simoncelli)
- WHAT: Systematic test of three multi-scale pyramid representations for coarse-to-fine phase correlation.
- WHY RELEVANT: Direct, evidence-based guidance for a design decision our own multi-scale pipeline (if adopted) would need to make.
- HOW IT COULD CONNECT/INTEGRATE: Adopt Gaussian pyramids as the default for stability, with awareness that Laplacian pyramids may help in strongly textured regions (useful for cratered highlands specifically) at some cost to noise robustness.
- WHAT WOULD NEED TO CHANGE: Validate against lunar imagery's specific noise/texture profile.
- ADVANTAGE: Removes guesswork on pyramid-type selection.
- CHALLENGE: Simoncelli pyramids need careful tuning and more computation for uncertain benefit — likely not worth the added complexity for us either, based on this evidence.
- OPPORTUNITY: Could inform a terrain-adaptive strategy — Gaussian pyramid as default, Laplacian pyramid selectively for high-relief/high-texture lunar regions (crater walls, highland terrain).

## 4. P2 — RESEARCH THEORY / TECHNICAL INSIGHT

### CONCEPT: Sub-pixel refinement strategy choice matters more than raw detector choice, but the two interact
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED] Among seven sub-pixel refinement strategies tested (paraboloid fitting, sinc interpolation, Gaussian peak fitting, centered-difference, Stone's frequency-domain method, FFT upsampling, Nelder–Mead, TPSS gradient), paraboloid fitting and TPSS were most reliable; but paraboloid fitting's best result (RMSE 0.010 px, 96.3% improvement) was achieved specifically in combination with the Fusion detector — the same refinement method paired with a weaker detector would not be expected to reach this result (per the paper's own conclusion that "performance depends on the joint optimization of detector–refinement combinations").
- WHY IT MATTERS: Warns against evaluating/optimizing sub-pixel refinement methods or detectors in isolation; the two must be co-tuned.
- PRACTICAL IMPLICATION: [INFERENCE] When we benchmark candidate matchers/detectors for lunar imagery, we should evaluate detector+refinement combinations jointly rather than ranking components independently.

### CONCEPT: Patch/window size trades off match density against geometric accuracy
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED] With 16×16 patches, all detectors yield many matches (SIFT: 7271 inliers) but higher RMSE (1.198–1.232 px); with 64×64 patches, Fusion achieves the lowest RMSE (1.103 px) and highest sub-pixel rate (56% below 1 px) but fewer total matches.
- WHY IT MATTERS: A direct, quantified illustration of the density-vs-accuracy trade-off relevant to the SIH "uniform distribution of matched points" requirement — larger patches give better individual-match accuracy but reduce overall point density/coverage.
- PRACTICAL IMPLICATION: [INFERENCE] Our own pipeline will face the same trade-off; a possible mitigation (untested here) is a multi-patch-size or adaptive-patch-size strategy — larger patches in low-texture regions, smaller patches where dense, distinctive texture is available.

### CONCEPT: Learned transformer matchers can produce geometrically invalid extrapolated matches
- WHAT THE PAPER ESTABLISHES: [REPORTED] When comparing LoFTR against the proposed hybrid method, "some correspondences fall outside the image domain," meaning the model extrapolates matches beyond valid spatial limits and "becomes unstable in some regions," despite LoFTR achieving the best vertical accuracy of any method tested.
- WHY IT MATTERS: A previously unreported (in Papers 2–3) failure mode for detector-free transformer matchers like LoFTR — directly relevant if we build on that architecture family.
- PRACTICAL IMPLICATION: [INFERENCE] Any LoFTR-based (or similar detector-free) component in our pipeline needs an explicit sanity check / bounds-validation step to catch and discard out-of-domain predicted correspondences, which classical geometric matching (as in this paper) does not appear to suffer from.

## 5. P3 — EXISTING / PAST SOLUTIONS

| APPROACH | PROBLEM ADDRESSED | CORE METHOD | RESULT | LIMITATION | RELEVANCE TO US |
|---|---|---|---|---|---|
| Rasmy et al. (2021) — prior work by same authors | VHR sub-pixel co-registration | Phase correlation + Harris detector (single-scale) | Baseline ("Original approach-Harris"): RMSEx=0.301m, RMSEy=0.304m (Pleiades) | No multi-scale, no Fusion detector, no systematic sub-pixel method comparison | P3 — shows incremental value of this paper's specific extensions (multi-scale, Fusion, sub-pixel refinement comparison) |
| Standard SIFT pipeline | General feature-based registration | SIFT detect+describe+match+RANSAC | Weakest results of all methods tested (RMSEx=0.516m, RMSEy=0.679m on Pleiades) | [DEMONSTRATED] "Descriptor matching alone is insufficient for high-precision registration" | P4 — reconfirms a now well-established weakness of pure classical feature pipelines |
| LoFTR (Paper 2's underlying method) | Detector-free 2D correspondence | Transformer-based coarse-to-fine matching | Best vertical accuracy in one axis (RMSEy=0.216m Pleiades) but worse horizontal (RMSEx=0.402m); unstable/out-of-domain matches in some regions | Imbalanced accuracy across axes; extrapolation failures | P0/P4 — important nuance to the otherwise positive picture of LoFTR from Papers 2–3 |
| This paper's hybrid method (Fusion + multi-scale phase correlation + paraboloid refinement + RANSAC) | VHR multi-temporal sub-pixel registration under terrain/radiometric variation | Combines detector, multi-scale phase correlation, sub-pixel refinement, RANSAC | Most stable/balanced accuracy across both axes and both datasets; RMSE < 0.2px (Sentinel-2), < 0.4px (Pleiades) | Same-sensor/same-modality only; classical (non-learned), so may not generalize as flexibly as learned methods to unseen conditions without retuning | P0 — strongest directly-applicable reference pipeline design found in this analysis so far |

## 6. P4 — WHAT NOT TO TRY / FAILURE MODES

### METHOD: Standard SIFT-only feature-based registration (no phase-correlation refinement)
- WHY PEOPLE USE IT: Simple, well-understood, widely available.
- FAILURE/WEAKNESS: [DEMONSTRATED] Weakest results across both test datasets (RMSEx=0.516m/5.028m, RMSEy=0.679m/5.148m for Pleiades/Sentinel-2 respectively) — worse than even the authors' simpler prior single-scale Harris-based baseline.
- WHY IT FAILS: Descriptor-based nearest-neighbor matching alone lacks the geometric/spectral refinement needed for true sub-pixel accuracy, consistent with findings across Papers 2–3 that classical NN-based matching underperforms context-aware or refinement-augmented approaches.
- CONDITIONS WHERE IT FAILS: VHR imagery with terrain-induced local distortions and radiometric differences between acquisitions.
- IMPLICATION FOR OUR PROJECT: Reconfirms (a third time, across three different papers/domains) that plain feature-based matching without additional refinement/context is inadequate for our sub-pixel accuracy requirement.
- POSSIBLE ALTERNATIVE: Any of the hybrid/context-aware/refinement-augmented approaches surveyed so far (this paper's hybrid pipeline, SuperGlue, LoFTR).

### METHOD: Blackman apodization window for phase correlation
- WHY PEOPLE USE IT: Strong sidelobe suppression, theoretically reduces spectral leakage most aggressively.
- FAILURE/WEAKNESS: [DEMONSTRATED] "Performs worst because it suppresses too much high-frequency information," reducing both accuracy and match density.
- WHY IT FAILS: Over-aggressive tapering removes the discriminative high-frequency structural content phase correlation needs to distinguish a true correlation peak.
- CONDITIONS WHERE IT FAILS: VHR imagery where fine surface texture/structure carries most of the discriminative signal — plausibly similar for lunar surface texture (small craters, boulders, regolith texture).
- IMPLICATION FOR OUR PROJECT: If we use any frequency-domain (phase correlation) component, avoid Blackman-style heavy windowing; prefer Gaussian/Tukey or no windowing, subject to our own validation.
- POSSIBLE ALTERNATIVE: Gaussian or Tukey apodization (best empirical trade-off reported here).

### METHOD: Detector-free transformer matching (LoFTR) without an out-of-domain sanity check
- WHY PEOPLE USE IT: Strong average accuracy, no dependency on hand-crafted detectors (see Paper 2/3 strengths).
- FAILURE/WEAKNESS: [REPORTED] Some correspondences fall entirely outside the valid image domain, meaning the model "extrapolates matches beyond valid spatial limits" and becomes locally unstable.
- WHY IT FAILS: Not explicitly diagnosed in this paper, but consistent with a coordinate-regression failure mode in learned dense/semi-dense matchers when confidence is misplaced.
- CONDITIONS WHERE IT FAILS: "In some regions" — not universally, but unpredictably, per the paper's account.
- IMPLICATION FOR OUR PROJECT: If we adopt a LoFTR-style component, we must add an explicit validity/bounds check and/or geometric consistency filter (e.g., RANSAC or DEGENSAC as in Paper 2) as a mandatory downstream safeguard, not an optional add-on.
- POSSIBLE ALTERNATIVE: Combine with classical geometric verification (as this paper and Paper 2 both do) rather than trusting raw learned-matcher output.

## 9. SOLUTION / DECISION MATRIX

| Finding / Method | What it solves | Why useful | Why NOT use it as-is | Challenge | Opportunity | Relevance |
|---|---|---|---|---|---|---|
| Full hybrid pipeline (Fusion + multi-scale phase correlation + sub-pixel + RANSAC) | Sub-pixel accuracy + outlier rejection + uniform-ish coverage, on real satellite imagery | Closest working analog to our target scenario; matches SIH's exact metric vocabulary | Same-sensor only; classical method may need retuning for lunar spectral/illumination extremes | Domain transfer to lunar imagery, esp. multi-sensor case | Strong reference-pipeline candidate, low training-data requirement | P0 |
| Fusion detector (multi-detector + geometric-structure filtering) | Reliable, well-distributed feature detection | Order-of-magnitude accuracy improvement demonstrated via ablation | Hough-line filter tuned for man-made structures, not craters | Needs a lunar-relevant structure filter (e.g., crater-rim detection) | High-value, low-risk adaptation | P1/P5 |
| RMSE/inlier-ratio/MedAE/%sub-pixel evaluation suite | Quantitative evaluation | Matches SIH-named metrics almost exactly | Needs reliable lunar GCPs/ground truth | Sourcing lunar ground-truth correspondence data | Ready-to-adopt evaluation template | P0 |
| Terrain-slope-stratified accuracy analysis | Realistic, terrain-aware accuracy characterization | Confirms geometry-driven accuracy ceiling in rugged terrain | N/A — an evaluation methodology, not a technique to implement, per se | Requires lunar slope/DTM data | Honest, defensible accuracy reporting; connects back to Paper 1's DTM co-registration | P1/P5 |
| Plain SIFT-only pipeline | Baseline general correspondence | N/A — confirmed weakest performer (3rd confirmation across papers) | Consistently underperforms all hybrid/learned alternatives | N/A | Avoid as primary strategy | P4 |
| Blackman apodization | Spectral leakage suppression | N/A — confirmed worst window choice | Over-suppresses discriminative high-frequency content | N/A | Avoid; use Gaussian/Tukey instead | P4 |
| LoFTR without bounds-checking | Detector-free matching | Good average accuracy | Can produce out-of-image-domain, unstable matches | Needs added geometric sanity-check layer | Confirms need for a verification stage in any LoFTR-based component | P4 |

## 10. KEY TAKEAWAYS FOR OUR PROJECT
1. This paper's hybrid classical pipeline is currently the closest, most directly transferable reference architecture found across all four papers analyzed — real satellite imagery, matching SIH's exact metrics, and no dependency on large learned-model training data.
2. The Fusion detector's core principle — dense keypoint generation filtered by proximity to geometrically stable structures — is a high-value, low-risk technique to adapt for lunar imagery, substituting crater/ridge structure detection for the paper's road/building-oriented Hough-line filter.
3. Adopt this paper's evaluation metric suite (RMSE, %<1px, %<0.5px, MedAE, inlier ratio, and an explicit sub-pixel-refinement "improvement" metric) as a template for our own SIH quantitative-evaluation deliverable.
4. New, important nuance on LoFTR (revising the optimistic picture from Papers 2–3): LoFTR can produce geometrically invalid, out-of-image-domain correspondences in some regions — any LoFTR-based component in our design needs a mandatory geometric sanity-check/verification stage, not an optional one.
5. Terrain-relief-driven accuracy limits are a first-class research finding, not just an evaluation footnote: in rugged terrain, accuracy becomes governed by imaging geometry (parallax, shadowing, occlusion) rather than the matching algorithm — directly relevant to lunar highland/crater-wall terrain, and motivates pairing our correspondence pipeline with DTM-based geometric correction (connecting back to Paper 1) for extreme-relief regions.
6. Confirmed (3rd time across papers) that plain descriptor-based feature matching (SIFT alone) is inadequate for sub-pixel accuracy requirements — do not consider this a viable standalone strategy.
7. Concrete, evidence-backed implementation defaults available if we adopt any phase-correlation component: Gaussian pyramid for multi-scale decomposition, Gaussian/Tukey apodization windows (avoid Blackman), paraboloid fitting for final sub-pixel refinement.
8. The paper's own stated future direction — combining classical geometric-stability methods with learned contextual matchers — mirrors an idea already emerging from our own cross-paper synthesis (Papers 2–4), reinforcing hybrid classical+learned architecture as a promising direction worth prioritizing in our solution design.
