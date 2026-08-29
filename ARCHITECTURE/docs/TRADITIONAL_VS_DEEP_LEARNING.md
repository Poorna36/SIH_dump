# Comparative Evaluation of Traditional and Deep Learning Feature Matching Algorithms using Chandrayaan-2 Lunar Data
(SIFT, ASIFT, AKAZE, RIFT2, SuperGlue — OHRC/IIRS/DFSAR vs NAC/WAC/SELENE)

Reference to SIH 26166: Multi-modal, Sun-angle and scale-invariant image correspondence (OHRC/TMC/IIRS)

---

## 1. RELEVANCE SNAPSHOT
- Overall relevance: Very High
- Most relevant component: Direct benchmark of 5 feature matching algorithms (3 classical + 1 radiation-insensitive + 1 deep learning) on the exact sensor pairs named in our problem statement (OHRC, IIRS) across equatorial and polar illumination conditions.
- Why it matters to OUR project: This is the closest paper yet to our actual task — it IS the correspondence/registration problem (not a side-task like radiometric mosaicking). It gives quantitative RMSE/timing benchmarks, a full preprocessing pipeline, and clear failure conditions for each algorithm family.
- Most valuable takeaway: [DEMONSTRATED] SuperGlue (deep learning) was the only algorithm that succeeded across all dataset pairs including polar regions and DFSAR-SELENE (radar-optical) cross-modality; all four classical/handcrafted methods (SIFT, ASIFT, AKAZE, RIFT2) failed outright on polar and SAR-SELENE datasets.

---

## 2. P0 — DIRECT CONNECTION / USE NOW

### Full Preprocessing Pipeline (Georeferencing → Resampling → Normalization → sensor-specific enhancement)
- WHAT: A staged pipeline: (1) georeferencing to align projections (Selenographic OHRC → Equirectangular NAC, etc.), (2) resolution resampling to bring source/reference to comparable GSD, (3) 8-bit intensity normalization, then (4) sensor-pair-specific enhancement.
- HOW IT WORKS: For OHRC–NAC: CLAHE → Inversion → Morphological Dilation → PCA. For IIRS–WAC: Histogram Matching → Shadow Normalization → Log Transformation. RIFT2/SuperGlue explicitly skip the specialized enhancement branch (per Fig.1 flowchart) — only the core pipeline (georeference/resample/normalize) is applied to them.
- WHY IT CONNECTS TO OUR REQUIREMENTS: Directly addresses our scale, multi-modal, and illumination robustness requirements as literal preprocessing steps before matching, using the *exact* sensors (OHRC, IIRS) named in our problem statement.
- WHAT WE COULD USE: The entire pipeline structure is directly reusable/adaptable as our own preprocessing baseline, with the branching logic (classical methods get heavy enhancement; RIFT2/SuperGlue get minimal preprocessing) as a validated design choice.
- Evidence from paper: [DEMONSTRATED] Fig. 1 methodology flowchart; Sections 4.1–4.2.
- Potential implementation consideration: Confirms that over-engineering preprocessing for the classical methods didn't rescue them in polar regions — investigate whether heavy preprocessing has diminishing/negative returns.

### SuperGlue (Deep learning matcher, GNN + optimal transport over SuperPoint features)
- WHAT: Graph neural network-based matcher that models spatial relationships between keypoints and solves an optimal transport problem for matching; typically paired with SuperPoint detector.
- HOW IT WORKS: Learns geometric priors from training data; leverages global image context rather than only local descriptors.
- WHY IT CONNECTS TO OUR REQUIREMENTS: Directly satisfies illumination robustness, scale robustness, viewpoint robustness, and multi-modal robustness — the four core challenges in our problem statement — simultaneously, as demonstrated on our exact target sensors.
- WHAT WE COULD USE: [DEMONSTRATED] Adopt SuperGlue (or successor architectures, e.g. LightGlue) as the core/primary correspondence engine, with classical methods (RIFT2 especially) potentially as fallback/ensemble.
- Evidence from paper: Table 2 (qualitative pass/fail grid) — SuperGlue is the only algorithm marked ✅ across all 6 dataset-region combinations (OHRC-NAC, IIRS-WAC, DFSAR-SELENE × equatorial/polar). Table 3 (quantitative RMSE/time) — SuperGlue achieves lowest or near-lowest RMSE in nearly every dataset pair, e.g. OHRC-NAC Equatorial RMSE_X 0.6249 / RMSE_Y 0.5718 vs SIFT's 3.6096/5.9558; and by far the fastest runtime (single-digit seconds vs hundreds of seconds for SIFT/ASIFT/AKAZE).
- Potential implementation consideration: [REPORTED] Paper notes SuperGlue's efficiency "is highly dependent on the quality and quantity of initial features (e.g., from SuperPoint) and requires domain-specific training for optimal results" — domain-specific (lunar) retraining/fine-tuning may be needed rather than off-the-shelf weights; GPU dependency for real-time use.

### RIFT2 (Radiation-insensitive Feature Transform 2, phase-congruency based)
- WHAT: Non-deep-learning method using phase congruency (local energy models, not intensity gradients) and a Maximum Index Map (MIM) descriptor, designed for cross-modal / illumination-variant matching.
- HOW IT WORKS: Rotation-invariance mechanism added over original RIFT; descriptors resilient to nonlinear radiometric variation since they don't rely on raw intensity gradients.
- WHY IT CONNECTS TO OUR REQUIREMENTS: Directly targets our illumination/multi-modal robustness requirement without needing training data — relevant as a non-deep-learning fallback or component of a hybrid/ensemble approach, especially if training data or compute is limited.
- WHAT WE COULD USE: [DEMONSTRATED] RIFT2 performed second-best after SuperGlue on OHRC-NAC Equatorial (RMSE 1.5033/1.1888, 36.9s) and worked on IIRS-WAC Polar (unlike SIFT/ASIFT/AKAZE which also technically ran there but RIFT2 succeeded qualitatively per Table 2). However it failed on OHRC-NAC Polar and DFSAR-SELENE Equatorial (Table 2 ❌).
- Evidence from paper: Table 2, Table 3, Section 5.1.2 (qualitative: "fewer, but more precise correspondences... particularly effective at handling radiometric differences").
- Potential implementation consideration: Worth testing as a lightweight, no-training-required component, but not reliable enough alone for polar/SAR conditions per this paper's results.

---

## 3. P1 — HIGHLY RELEVANT / POTENTIALLY INTEGRABLE

### RANSAC-based outlier rejection + homography estimation
- WHAT: Standard RANSAC applied after feature matching to filter outliers and compute a 3×3 homography for warping.
- WHY RELEVANT: Our problem statement explicitly requires "outlier rejection" and "accurate geometric registration" — RANSAC is the standard tool for both, confirmed effective here.
- HOW IT COULD INTEGRATE: Directly reusable as the outlier-rejection/registration stage after any feature-matching front end (classical or SuperGlue).
- WHAT WOULD NEED TO CHANGE: None structurally; may need tuning of inlier thresholds per sensor pair given very different point-count/RMSE regimes (e.g., DFSAR-SELENE has very few matches per Fig. 11).
- ADVANTAGE: [REPORTED] "Crucial... effectively handled the high proportion of outliers that typically occur due to the repetitive nature of lunar features, illumination differences, and sensor variations."
- CHALLENGE: Not evaluated independently in this paper — its contribution vs. raw matcher output isn't isolated in the metrics.
- OPPORTUNITY: Could add a uniform-spatial-distribution constraint on top of standard RANSAC to directly satisfy our problem statement's explicit "uniform distribution of matched points" requirement, which this paper does not address at all (see P6/gap).

### Coordinate System Integration step (post-warp georeferencing reconciliation)
- WHAT: After RANSAC + warping, the warped image's coordinate system is re-aligned to the reference's original CRS using metadata-derived top-left coordinates.
- WHY RELEVANT: Needed for a "registered product" deliverable as required by our problem statement (not just a warped raster, but a scientifically usable geo-registered output).
- HOW IT COULD INTEGRATE: Reusable as the final step of a registration pipeline for deliverable generation.
- WHAT WOULD NEED TO CHANGE: Depends on specific metadata formats of OHRC/TMC/IIRS PDS-style products — needs verification against actual ISSDC data format.
- ADVANTAGE: Produces scientifically valid, directly comparable geo-registered products.
- CHALLENGE: Implementation detail-heavy, tied to specific projections (Selenographic vs Equirectangular).
- OPPORTUNITY: Directly reusable engineering component, low novelty risk.

---

## 4. P2 — RESEARCH THEORY / TECHNICAL INSIGHT

### CONCEPT: Preprocessing effort does not compensate for a fundamentally intensity/gradient-based descriptor's limits
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED] Despite heavy specialized preprocessing (CLAHE, inversion, dilation, PCA for OHRC-NAC; histogram matching, shadow normalization, log transform for IIRS-WAC), SIFT/ASIFT/AKAZE still failed completely on polar-region data (Table 2, all ❌ for polar OHRC-NAC and DFSAR-SELENE both regions).
- WHY IT MATTERS: Directly answers a design question for our project: is it enough to fix illumination robustness via preprocessing + classical descriptors? This paper's evidence says no for extreme lunar polar conditions.
- PRACTICAL IMPLICATION: Preprocessing (CLAHE, shadow correction, etc.) helps classical methods in equatorial/moderate conditions but is insufficient alone for polar or cross-modal (SAR) extremes — a learning-based or phase-congruency-based matcher is needed for full requirement coverage.

### CONCEPT: "No preprocessing needed" methods (RIFT2, SuperGlue) outperform heavily preprocessed classical methods
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED] "algorithms that do not necessitate explicit preprocessing can indeed outperform traditional methods that do, both in terms of accuracy and computational efficiency" (Section 5.2).
- WHY IT MATTERS: Suggests preprocessing-heavy pipelines may be a false economy — an important architectural insight for our project's design trade-off between preprocessing complexity and matcher robustness.
- PRACTICAL IMPLICATION: [INFERENCE] Consider prioritizing engineering effort on a robust matcher (deep learning or phase-congruency) over an elaborate preprocessing pipeline, though preprocessing still matters for classical baselines/ensembles.

### CONCEPT: Polar lunar imagery is categorically harder — near-total failure zone for classical methods
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED] All four non-deep-learning methods failed on OHRC-NAC Polar and DFSAR-SELENE (both regions); only SuperGlue succeeded (Table 2). Paper attributes this to "significant viewpoint changes and variations in crater shapes" combined with low solar angles/permanent shadow.
- WHY IT MATTERS: Confirms polar illumination is not just "harder" but a qualitatively different failure regime — relevant if our project's evaluation data includes polar OHRC/TMC/IIRS scenes (per our problem statement's general Sun-angle robustness requirement).
- PRACTICAL IMPLICATION: If evaluation data includes polar scenes, a deep-learning matcher (or something with similar global-context reasoning) is likely mandatory, not optional.

---

## 5. P3 — EXISTING / PAST SOLUTIONS

| APPROACH | PROBLEM ADDRESSED | CORE METHOD | RESULT | STRENGTH | LIMITATION | RELEVANCE TO US |
|---|---|---|---|---|---|---|
| SIFT | General scale/rotation-invariant matching | DoG scale-space, 128-D gradient descriptor | [DEMONSTRATED] Best on OHRC-NAC/IIRS-WAC equatorial (moderate RMSE) but 0 successes on polar/SAR-SELENE | Well-understood, no training | High compute (678–809s for OHRC-NAC); fails under viewpoint/illumination extremes; degrades in low-feature regions | High — direct baseline for our project, but confirmed inadequate alone |
| ASIFT | Affine/viewpoint invariance via simulated tilts | Multiple SIFT runs over simulated affine views | [DEMONSTRATED] Slightly better than SIFT on OHRC-NAC Eq. (RMSE 1.99/1.63) but still fails polar/SAR | Better viewpoint robustness than SIFT | ~2x+ compute cost of SIFT (809.8s); still fails polar & SAR-SELENE | Medium — improvement direction confirmed insufficient for our hardest cases |
| AKAZE | Fast, real-time-capable local features | Nonlinear scale space (Perona-Malik), M-LDB binary descriptor | [DEMONSTRATED] Fast on IIRS-WAC (0.16–0.23s) and decent equatorial accuracy; fails polar & SAR | Fast, lightweight | Many "high-count but misaligned/outlier-prone" matches (qualitative finding, 5.1.1); fails polar & SAR-SELENE | Medium — speed advantage possibly useful in hybrid/ensemble, but reliability concerns noted |
| RIFT2 | Cross-modal / illumination-insensitive matching | Phase congruency + Maximum Index Map descriptor | [DEMONSTRATED] 2nd-best accuracy on OHRC-NAC Eq. (1.50/1.19, 36.9s); succeeds IIRS-WAC both regions; fails OHRC-NAC Polar & SAR-SELENE Eq. | No training required, illumination-insensitive by design | Struggles with severe geometric distortion / low-feature areas (per lit. review); inconsistent across dataset types | High — best non-DL option, but not universally reliable per this paper's own results |
| SuperGlue | Deep-learning-based robust matching across all conditions | GNN + optimal transport over SuperPoint keypoints | [DEMONSTRATED] Only method succeeding on all 6 dataset/region combinations; lowest RMSE almost everywhere; fastest runtime | Global context reasoning; handles polar & SAR-optical cross-modal cases | Requires GPU, quality initial features, and "domain-specific training for optimal results" (unclear if authors fine-tuned or used generic weights) | Very High — current best candidate as core matcher for our project |

---

## 6. P4 — WHAT NOT TO TRY / FAILURE MODES

### METHOD: SIFT / ASIFT / AKAZE for polar lunar imagery
- WHY PEOPLE USE IT: Well-established, no training data required, widely available implementations, works well on Earth/general imagery and even lunar equatorial imagery.
- FAILURE / WEAKNESS: [DEMONSTRATED] Complete registration failure (marked ❌, "NA" for RMSE/time) on OHRC-NAC Polar dataset — could not even produce usable output despite full preprocessing pipeline.
- WHY IT FAILS: [REPORTED/INFERENCE] Extreme low solar angles and permanent-shadow regions at the poles cause severe nonlinear intensity/contrast changes and viewpoint-driven crater-shape distortion that intensity-gradient-based descriptors (SIFT-family) cannot handle, even after CLAHE/inversion/dilation/PCA enhancement.
- CONDITIONS WHERE IT FAILS: Polar lunar regions with low Sun elevation and heavy shadowing; also fails on cross-modal SAR–optical pairs (DFSAR-SELENE) regardless of region.
- IMPLICATION FOR OUR PROJECT: If our evaluation includes polar or SAR-like extreme illumination scenes, classical SIFT-family methods should not be relied upon as the primary/sole matcher.
- POSSIBLE ALTERNATIVE: SuperGlue (or similar learned matcher) or RIFT2 for cross-modal cases (though RIFT2 also failed on OHRC-NAC Polar specifically).

### METHOD: RIFT2 for DFSAR-SELENE (SAR-optical) and OHRC-NAC Polar
- WHY PEOPLE USE IT: Designed specifically for cross-modal/radiation-insensitive matching, theoretically well-suited for SAR-optical and illumination-variant pairs.
- FAILURE / WEAKNESS: [DEMONSTRATED] Despite its illumination-insensitive design, RIFT2 still failed (❌) on OHRC-NAC Polar and DFSAR-SELENE Equatorial (Table 2).
- WHY IT FAILS: [INFERENCE — not fully explained by paper] Likely due to a combination of severe geometric/viewpoint distortion (not addressed by phase congruency alone) and/or insufficient distinctive structure in the specific SAR-optical pairing; paper explicitly states "further investigation is needed."
- CONDITIONS WHERE IT FAILS: Extreme geometric distortion combined with radar-optical cross-modality; polar OHRC-NAC specifically.
- IMPLICATION FOR OUR PROJECT: Don't assume any single "illumination-robust" classical method is a complete solution for all combinations of our stated challenges (illumination + viewpoint + multi-modal simultaneously) — RIFT2 handles illumination/modality but apparently not combined with severe geometric distortion.
- POSSIBLE ALTERNATIVE: SuperGlue succeeded on both these specific failure cases — reinforces it as the most complete single option in this paper's test set.

### METHOD: AKAZE's high match-count as a proxy for quality
- WHY PEOPLE USE IT: More matches often assumed to mean more robust/accurate registration.
- FAILURE / WEAKNESS: [REPORTED, qualitative] AKAZE "consistently identifies a large number of keypoints and matches... despite the high match count, several of these matches appear misaligned or loosely clustered around non-overlapping regions."
- WHY IT FAILS: High recall of keypoints does not imply high precision/distinctiveness — many matches are outliers or from repetitive, low-texture lunar terrain.
- CONDITIONS WHERE IT FAILS: Repetitive/uniform lunar terrain lacking strongly distinctive local structure (a challenge explicitly named in our problem statement).
- IMPLICATION FOR OUR PROJECT: Match count alone is not a reliable accuracy metric; rely on RMSE / inlier ratio (as our problem statement itself specifies) rather than raw match counts.
- POSSIBLE ALTERNATIVE: Always report RMSE + inlier ratio + uniformity of spatial distribution together, not match count alone.

---

## 9. SOLUTION / DECISION MATRIX

| Finding / Method | What it solves | Why useful | Why NOT use it | Challenge | Opportunity | Relevance |
|---|---|---|---|---|---|---|
| SuperGlue | Robust correspondence across illumination/viewpoint/scale/multi-modal variation | Only method succeeding on all 6 test conditions incl. polar & SAR-optical; best RMSE & speed | Needs GPU; may need domain-specific fine-tuning for optimal sub-pixel results | Requires good initial keypoints (SuperPoint) and possibly lunar-specific training data | Adopt as primary matcher; fine-tune on OHRC/TMC/IIRS pairs | P0 |
| RIFT2 | Illumination/multi-modal-insensitive matching, no training | Best non-DL option; reasonably fast; no training data needed | Fails on severe geometric distortion (OHRC-NAC Polar) and SAR-SELENE Eq. | Root cause of failure unclear ("further investigation needed" per authors) | Useful fallback/no-GPU option or ensemble component | P1/P3 |
| SIFT/ASIFT/AKAZE + heavy preprocessing (CLAHE, shadow norm, etc.) | Baseline equatorial optical registration | Well-understood, interpretable | Complete failure in polar & SAR-optical; high compute cost | Cannot be fixed by preprocessing alone (demonstrated) | Use only as equatorial-condition baseline/ensemble member, not primary solution | P3/P4 |
| RANSAC + homography + CRS reconciliation | Outlier rejection & final registered product generation | Standard, proven, satisfies deliverable requirement | N/A | Doesn't address spatial-uniformity requirement | Add uniformity-enforcement post-processing | P1/P5 |
| Full preprocessing pipeline (georeference→resample→normalize→sensor-specific enhancement) | Standardizes heterogeneous sensor inputs before matching | Directly matches our sensor set (OHRC, IIRS) | Doesn't rescue classical methods in extreme conditions | Sensor-specific branches need adaptation for TMC (untested here) | Reusable pipeline skeleton; adapt enhancement branch for TMC | P0/P2 |

---

## 10. KEY TAKEAWAYS FOR OUR PROJECT
1. [DEMONSTRATED] SuperGlue is currently the strongest single candidate for our core correspondence engine — it is the only method in this direct benchmark (on our exact sensor types, OHRC & IIRS) that succeeds across equatorial, polar, and cross-modal (SAR-optical) conditions simultaneously.
2. [DEMONSTRATED] Classical methods (SIFT/ASIFT/AKAZE) fail completely at the poles even with heavy specialized preprocessing — preprocessing cannot substitute for a fundamentally more robust matching approach under extreme Sun-angle/shadow conditions.
3. RIFT2 is the best non-deep-learning fallback but is not reliable enough alone for the hardest cases (polar OHRC-NAC, SAR-SELENE) — its failure mechanism is unexplained in the literature and flagged by the authors themselves as needing further study.
4. A directly reusable preprocessing pipeline exists (georeferencing → resampling → 8-bit normalization → sensor-specific enhancement) tailored to OHRC-NAC and IIRS-WAC pairs — adaptable starting point for our pipeline, though TMC-specific preprocessing is untested and would need to be developed.
5. Critical unaddressed gap in this paper (and thus an opportunity for us): the paper never addresses the "uniform spatial distribution of matched points" requirement that our problem statement explicitly demands — we should add this as a distinct post-processing step (e.g., grid-based match selection).
6. Match count is not a reliable quality proxy (AKAZE case) — always evaluate via RMSE + inlier ratio + spatial uniformity together, consistent with our problem statement's own suggested metrics.
7. TMC/TMC-2 (one of our three named primary sensors) was not evaluated in this paper — only OHRC, IIRS, DFSAR were tested. This is a direct research/data gap for our specific project scope.
8. Sub-pixel accuracy target is nearly but not fully met: best SuperGlue RMSE values (~0.4–0.9 px) are close to sub-pixel but not consistently below 1.0 px across all conditions — domain-specific fine-tuning (P5) may be needed to reliably hit our problem statement's sub-pixel accuracy requirement.

---
