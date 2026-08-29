# LoFTR Submission to Image Matching Challenge 2021 (He, Wang, Sun et al.)

## 1. RELEVANCE SNAPSHOT
- Overall relevance: High
- Most relevant component: Detector-free, coarse-to-fine, sub-pixel semi-dense image matching (LoFTR), plus its point-density/uniformity control (greedy merging) and outlier rejection (DEGENSAC).
- Why it matters to OUR project: This is a genuine 2D image-correspondence pipeline — unlike Paper 1 — with a built-in coarse-to-fine sub-pixel refinement stage, semi-dense/uniformly-distributed matches, and geometric outlier rejection, which map almost one-to-one onto SIH 26166's stated requirements (sub-pixel accuracy, uniform match distribution, outlier rejection).
- Most valuable takeaway: Detector-free matching avoids depending on repeatable keypoint *detection* across views (a common failure point under Sun-angle/illumination change), while still needing a companion strategy (LoFTR-SPP + greedy merging) to produce a fixed, spatially-controlled point set — a pattern directly adaptable to our multi-sensor lunar correspondence problem. Note: this is a short technical-details/submission report, not the full LoFTR architecture paper — deeper architectural detail should be pulled from the original LoFTR CVPR 2021 paper if needed.

## 2. P0 — DIRECT CONNECTION / USE NOW

### LoFTR coarse-to-fine, detector-free semi-dense matching
- WHAT: Produces matches directly between two images without pre-extracted keypoints; coarse matches from regular grid points on coarse feature maps, then right-side matches refined to sub-pixel level using fine-level feature maps.
- HOW IT WORKS: Transformer-based feature matching conditioned jointly on both images (cross-image attention), avoiding independent single-image keypoint detection.
- WHY IT CONNECTS TO OUR REQUIREMENTS: Directly satisfies the SIH requirement for sub-pixel accuracy; detector-free design is inherently more robust to appearance change than classical detect-then-describe pipelines, relevant to Sun-angle/illumination variation.
- WHAT WE COULD USE: The coarse-to-fine refinement concept as the backbone matching strategy, or as a refinement module bolted onto another coarse matcher.
- Evidence from paper: [REPORTED] Coarse matches from grid points; matches refined "to a sub-pixel level using fine-level feature maps and a coarse-to-fine module."
- Potential implementation consideration: Requires training data with realistic covisibility/overlap ranges for lunar imagery (paper's own training used MegaDepth natural-photo pairs — domain gap to lunar terrain, see P4/P5).

### Grid-based, evenly-distributed left-view matches
- WHAT: LoFTR by default always uses evenly distributed grid points as the left-image matches (before the LoFTR-SPP modification).
- WHY IT CONNECTS: Directly addresses the SIH requirement for a uniform distribution of matched points across the source image.
- WHAT WE COULD USE: Grid-conditioned coarse matching as a mechanism to guarantee spatial coverage, rather than relying purely on sparse, unevenly-distributed detected keypoints (e.g., craters only in some regions).
- Evidence from paper: [REPORTED] LoFTR "always uses evenly distributed grid points as left matches."
- Potential implementation consideration: Pure grid points are good for coverage but hinder multi-view consistency (see P1, P4) — a trade-off to manage.

### DEGENSAC-based geometric verification
- WHAT: Custom geometric verification using DEGENSAC (degenerate-configuration-aware RANSAC variant), max 10,000 iterations, 0.99999 confidence threshold.
- WHY IT CONNECTS: Directly addresses the SIH requirement for outlier rejection in correspondences.
- WHAT WE COULD USE: DEGENSAC (or standard RANSAC/MAGSAC variants) as the final outlier-filtering stage before computing registration/homography, especially relevant since lunar images can contain large near-planar regions (mare, crater floors) where standard RANSAC degenerates.
- Evidence from paper: [REPORTED] Uses DEGENSAC with the stated iteration/confidence parameters.
- Potential implementation consideration: DEGENSAC specifically guards against dominant-plane degeneracy — relevant for flat lunar terrain regions, a plausible failure mode worth testing explicitly.

## 3. P1 — HIGHLY RELEVANT / POTENTIALLY INTEGRABLE

### LoFTR-SPP: hybrid detector-free + pre-extracted keypoints (SuperPoint)
- WHAT: Augments LoFTR by replacing a left-view grid point with a nearby pre-extracted SuperPoint keypoint (within a window) when available, then refining the right-view match toward that keypoint instead of the raw grid point.
- WHY RELEVANT: Solves the "fixed keypoint set per image" requirement that pure detector-free matching lacks — same practical constraint we'd face needing a stable point set usable across OHRC/TMC/IIRS multi-view/multi-sensor combinations.
- HOW IT COULD CONNECT/INTEGRATE: Could pair a detector-free coarse matcher with a lunar-domain-adapted keypoint detector (rather than generic SuperPoint) to get both cross-view consistency and detector-free robustness.
- WHAT WOULD NEED TO CHANGE: SuperPoint would need retraining/fine-tuning on lunar optical imagery (it is trained on natural images); need to validate consistency across illumination/sensor differences, not just viewpoint.
- ADVANTAGE: Improves feature repeatability across multiple views (useful for multi-image lunar mosaicking/registration, not just pairwise).
- CHALLENGE: Adds a second network dependency (detector) and window-matching heuristics.
- OPPORTUNITY: Could be the basis of a "detector-free primary matcher + domain keypoint detector for consistency" hybrid architecture for OHRC↔TMC/IIRS/LRO-NAC matching.

### Greedy points merging via adaptive NMS (bisection-searched suppression radius)
- WHAT: Collects all candidate points per image (from all pairwise matches), performs NMS with a suppression radius adaptively tuned via bisection search to hit a target point-count budget (e.g., 8k points), merging nearby suppressed points into surviving "pillar points," with matches updated accordingly.
- WHY RELEVANT: A concrete, tunable mechanism for controlling point density/count while preserving spatial spread — directly useful for enforcing SIH's "uniform distribution" constraint under a match-count budget.
- HOW IT COULD CONNECT/INTEGRATE: Could be applied as a post-processing step on any coarse+fine matcher's raw output to produce a clean, budget-constrained, spatially-distributed match set for evaluation (RMSE/inlier count/inlier ratio).
- WHAT WOULD NEED TO CHANGE: Threshold/radius tuning would need to be re-validated against lunar image resolution/scale ranges.
- ADVANTAGE: Principled, score-aware point reduction rather than naive random subsampling.
- CHALLENGE: Adds pipeline complexity; conflicting one-to-many merges are explicitly not handled by the authors ("could be further filtered and selected but is actually not handled").
- OPPORTUNITY: A cleaner conflict-resolution step here could be a genuine, implementable improvement in our project relative to this reported baseline.

## 4. P2 — RESEARCH THEORY / TECHNICAL INSIGHT

### CONCEPT: Detector-free matching trades single-view feature consistency for match quality
- WHAT THE PAPER ESTABLISHES: [REPORTED] Detector-free methods (LoFTR, NCNet, Truong et al.'s dense correspondence method) do not rely on independently-detected, per-image keypoints, but consequently "lack consistent features within a single image" — fine for relative pose estimation, but a problem for tasks needing a fixed, reusable keypoint set (e.g., SfM, or multi-image benchmarks like IMC).
- WHY IT MATTERS: SIH 26166's deliverable includes reusable "corresponding match points," implying some need for consistent, identifiable points — the same tension this paper explicitly navigates.
- PRACTICAL IMPLICATION: [INFERENCE] If our deliverable needs a stable point set (e.g., ground-control-point style workflows, repeated cross-sensor registration), a pure detector-free approach alone is insufficient; a hybrid design (as in LoFTR-SPP) is likely necessary.

### CONCEPT: Training data covisibility range determines what viewpoint/overlap conditions a matcher generalizes to
- WHAT THE PAPER ESTABLISHES: [REPORTED] Training used MegaDepth image pairs restricted to covisibility scores in [0.1, 0.7], split into finer sub-ranges [0.1,0.3], [0.3,0.5], [0.5,0.7] across 368 sub-scenes, trained from scratch (no pretrained backbone).
- WHY IT MATTERS: Shows matcher performance is tied to the overlap/viewpoint distribution seen during training — not domain- or condition-agnostic by default.
- PRACTICAL IMPLICATION: [INFERENCE] A LoFTR-style matcher trained only on MegaDepth (natural, Earth-based, single-sensor, daylight-consistent photos) has no exposure to Sun-angle/illumination extremes, multi-sensor spectral differences, or lunar low-texture terrain — a significant domain gap for our use case (see P4/P5). Adoption would need lunar/multi-sensor-specific retraining or synthetic data generation.

## 5. P3 — EXISTING / PAST SOLUTIONS

| APPROACH | PROBLEM ADDRESSED | CORE METHOD | RESULT | LIMITATION | RELEVANCE TO US |
|---|---|---|---|---|---|
| NCNet (Rocco et al., 2018) | Dense correspondence w/o detectors | Neighbourhood consensus network | Cited as predecessor detector-free method | Not detailed in this report | P6 — background lineage only |
| Truong et al. (2021) | Dense correspondence + confidence estimation | Learned dense matching w/ trust/confidence scores | Cited as related detector-free method | Not detailed in this report | P5 — confidence-aware matching relevant to outlier rejection, worth separate follow-up |
| Base LoFTR (Sun et al., CVPR 2021) | General image matching | Transformer-based coarse-to-fine detector-free matching | Basis of this submission | Architecture not detailed in this short report | P0 — recommend reading full paper for architecture depth |
| This submission (LoFTR-SPP + greedy merging + DEGENSAC) | IMC 2021 benchmark constraints (fixed keypoint sets) | LoFTR + SuperPoint hybrid + adaptive NMS merging + DEGENSAC verification | Competitive IMC submission (accuracy numbers not in this report) | Domain: natural outdoor photos (MegaDepth); one-to-many merge conflicts unresolved | P0/P1 — most directly transferable engineering pattern found so far |

## 6. P4 — WHAT NOT TO TRY / FAILURE MODES

### METHOD: Pure grid-based detector-free matching without any consistency mechanism
- WHY PEOPLE USE IT: Simple, guarantees uniform spatial coverage, avoids keypoint-detector failure modes.
- FAILURE/WEAKNESS: [REPORTED] "Hinders the performance of SfM because of inconsistent features within a single view and unrepeatable features among multiple views."
- WHY IT FAILS: Grid points are arbitrary image locations, not tied to salient/repeatable structures, so the same physical point isn't guaranteed to be re-selected across different images of the same scene.
- CONDITIONS WHERE IT FAILS: Tasks requiring the *same* physical point to be re-identified across 3+ images or multiple sensors — precisely our OHRC/TMC/IIRS multi-sensor scenario.
- IMPLICATION FOR OUR PROJECT: If our pipeline needs consistent tie points across more than a pair of images/sensors, pure grid-based detector-free matching alone is inadequate.
- POSSIBLE ALTERNATIVE: Hybridize with a domain-adapted keypoint detector (as in LoFTR-SPP), or enforce consistency via multi-view refinement.

### METHOD: Naive/unhandled one-to-many match merging
- WHY PEOPLE USE IT: Greedy NMS merging is fast and simple under a point-count budget.
- FAILURE/WEAKNESS: [REPORTED] Authors acknowledge conflicting one-to-many matches from merging are not resolved in their submission.
- WHY IT FAILS: Merging nearby points from independent pairwise matches can create ambiguous correspondences without resolution logic.
- CONDITIONS WHERE IT FAILS: Any downstream step assuming one-to-one correspondence integrity (precise RMSE, bundle adjustment).
- IMPLICATION FOR OUR PROJECT: If we adopt a similar merging strategy, we should design conflict resolution explicitly, unlike this baseline.
- POSSIBLE ALTERNATIVE: Keep highest-confidence match on conflict, or resolve via small local optimization.

## 9. SOLUTION / DECISION MATRIX

| Finding / Method | What it solves | Why useful | Why NOT use it as-is | Challenge | Opportunity | Relevance |
|---|---|---|---|---|---|---|
| LoFTR coarse-to-fine detector-free matching | Sub-pixel, semi-dense correspondence | Matches core SIH sub-pixel + coverage needs | Trained on natural photos; no illumination/multi-sensor robustness built-in | Needs lunar-domain retraining | Strong candidate backbone, pending domain adaptation | P0 |
| LoFTR-SPP (+ SuperPoint) | Cross-view/multi-image point consistency | Needed for multi-sensor/multi-view tie points | SuperPoint untrained on lunar imagery | Retrain detector on lunar data | Hybrid detector-free + domain-detector architecture | P1 |
| Greedy adaptive-NMS point merging | Uniform match distribution under a point budget | Directly reusable, matcher-agnostic | One-to-many conflicts unresolved in original | Needs conflict-resolution logic added | Ready-to-adapt post-processing module | P1/P5 |
| DEGENSAC geometric verification | Outlier rejection, handles dominant-plane degeneracy | Matches SIH outlier-rejection requirement; relevant to flat lunar terrain | N/A — good candidate as-is | Parameter tuning for our data | Use directly as final-stage outlier filter | P0 |
| Pure grid-based left points (no hybrid) | Uniform coverage | Simple | Not consistent across views/sensors | N/A | Avoid as sole strategy; combine with detector | P4 |

## 10. KEY TAKEAWAYS FOR OUR PROJECT
1. LoFTR-style detector-free, coarse-to-fine, transformer-based matching is currently the strongest architectural candidate found for meeting sub-pixel accuracy and uniform-distribution requirements simultaneously — but needs lunar/multi-sensor-specific training data, not just off-the-shelf weights.
2. DEGENSAC (or a similar degeneracy-aware RANSAC) should be adopted as our outlier-rejection stage, given large flat/near-planar terrain regions on the Moon.
3. The greedy adaptive-NMS point-merging module is directly reusable and matcher-agnostic for enforcing a spatially uniform, budget-constrained match set — cheap to implement, worth prioritizing early.
4. Key domain/research gap: no evidence of robustness to Sun-angle/illumination change or cross-sensor (multi-modal) matching — MegaDepth training only covers natural-image viewpoint/overlap variation. Must be sourced from other papers or solved via our own domain-specific training/data augmentation.
5. Avoid pure grid-based detector-free matching without a consistency mechanism if multi-view/multi-sensor tie-point reuse is required — a confirmed weakness per the paper's own account.
6. Unresolved conflict handling in point-merging is a known gap we could improve on relative to this reported baseline — a concrete, low-risk implementation opportunity.
7. Consider Truong et al.'s confidence-aware dense correspondence method (cited but not detailed here) as a follow-up paper — directly relevant to combining match confidence with outlier-rejection/uniform-distribution needs.
