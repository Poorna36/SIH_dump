# Robust Feature Matching of Multi-Illumination Lunar Orbiter Images Based on Crater Neighborhood Structure (CNSFM)
(Crater detection + geometric-topology matching — LROC NAC, evaluated across equatorial/mid-latitude/polar scenes)

Reference to SIH 26166: Multi-modal, Sun-angle and scale-invariant image correspondence (OHRC/TMC/IIRS)

---

## 1. RELEVANCE SNAPSHOT
- Overall relevance: Very High
- Most relevant component: A complete, purpose-built illumination-invariant correspondence method (CNSFM) that abandons grayscale/gradient descriptors entirely in favor of crater-geometry structure, directly targeting the exact failure mode (Sun-angle/shadow variation) named as our #1 challenge.
- Why it matters to OUR project: This paper directly and quantitatively demonstrates a working, superior alternative for the hardest case in our problem statement — polar, high-Sun-azimuth-difference lunar imagery — where Paper 2's classical methods (SIFT/ASIFT/AKAZE/RIFT2) all failed completely.
- Most valuable takeaway: [DEMONSTRATED] By using crater centers + their geometric neighborhood topology (not pixel intensity/gradients) as the matching unit, CNSFM achieved 100% success rate in equatorial/mid-latitude regions and 72.3% in extreme polar shadow conditions — versus a best of 31.2% (WSSF) among all baselines (SIFT, HAPCG, ML-HLMO, WSSF) at the pole.

---

## 2. P0 — DIRECT CONNECTION / USE NOW

### Crater Neighborhood Structure Feature Matching (CNSFM) — full pipeline
- WHAT: A four-stage pipeline: (1) deep-learning crater detection (YOLOv9, transfer-learned on a custom lunar crater dataset), (2) Crater Neighborhood Structure Feature (CNSF) construction — a central crater + its K-nearest-neighbor craters, (3) CNSF similarity matching using similarity-transformation-invariant geometric parameters (angles, side-length ratios, diameter ratios), (4) outlier removal via local geometric transformation consistency.
- HOW IT WORKS: Instead of matching individual pixel patches/gradient descriptors (which break under nonlinear illumination change), the method matches geometric topology between craters — angles and distance ratios that are invariant under similarity transformation (scale/rotation/translation) and, critically, invariant under illumination change since they depend only on crater *center positions and diameters*, not pixel intensity.
- WHY IT CONNECTS TO OUR REQUIREMENTS: Directly and simultaneously addresses illumination/Sun-angle robustness (core motivation), scale robustness (built into similarity-invariant math and explicit scale-constraint step), and outlier rejection (dedicated Mismatched CNSF Removal stage) — three of our four core challenges in a single unified method.
- WHAT WE COULD USE: [DEMONSTRATED] The entire CNSFM approach is directly reusable as a candidate primary or complementary correspondence method for OHRC/TMC (panchromatic, crater-rich) imagery. Authors have released code + dataset publicly.
- Evidence from paper: Table 1 (Success Rate): CNSFM achieves 100% SR in Scenes S1 (mid-latitude) and S2 (equatorial), and 72.3% SR in Scene S3 (near south pole, <2° incidence-angle range but up to 180° azimuth difference) — vs. SIFT 17.3%, HAPCG 22.1%, ML-HLMO 13.9%, WSSF 31.2% in S3. Table 3 (RCM — reliability of matches): CNSFM reaches 100% RCM in S1/S3 and 99.3% in S2, the highest of all methods.
- Potential implementation consideration: Requires (a) a reliable crater detector for our specific sensors (OHRC/TMC/IIRS), which would likely need retraining/fine-tuning since the released detector is trained only on LROC NAC imagery at 0.5 m resolution; (b) sufficient crater density in the scene — explicitly fails in crater-sparse young geological units (mare/impact melt ponds) — a genuine limitation for some lunar terrain types.
- Code/data availability: [REPORTED] Both dataset and executable are publicly available at https://github.com/Bin501/CNSFM — directly reusable/testable resource for our project.

### Similarity-invariant geometric distance measure for illumination-independent matching
- WHAT: A formal distance metric (Eq. 10) between two Crater Neighborhood Structures, built from angle/ratio vectors that are provably invariant under any similarity transformation (scale + rotation + translation), matched via Nearest Neighbor Distance Ratio (NNDR).
- HOW IT WORKS: Because craters are represented purely by (x, y, diameter) — not pixel intensities — the descriptor is mathematically illumination-invariant by construction, not merely "illumination-robust" via engineering (as with RIFT2/CLAHE-style approaches in Papers 1–2).
- WHY IT CONNECTS TO OUR REQUIREMENTS: This is a fundamentally different strategy than every method in Papers 1 & 2 — it sidesteps the illumination problem entirely rather than trying to normalize or be robust to it.
- WHAT WE COULD USE: The mathematical framework (similarity invariants of angle/distance-ratio triples) is directly reusable/adaptable, and could in principle be extended to other stable geometric landmarks beyond craters (e.g., ridges, boulder fields) if crater density is insufficient in a given scene.
- Evidence from paper: Eq. (4), (7), (10); ablation-style Figure 9 shows 138/241 (57.3%) and 138/201 (68.6%) crater re-detection rate under a 41.1° incidence + 76.7° azimuth difference, i.e., core geometric primitives (crater positions) survive extreme illumination change even when pixel appearance does not.
- Potential implementation consideration: Requires accurate sub-pixel crater center localization for downstream registration accuracy; paper doesn't report standalone crater-localization accuracy in physical units, only downstream RMSE (~1.0–2.2 px average).

---

## 3. P1 — HIGHLY RELEVANT / POTENTIALLY INTEGRABLE

### YOLOv9-based crater detector via transfer learning (custom-labeled dataset)
- WHAT: YOLOv9-C (PGI + GELAN architecture) fine-tuned on a manually-annotated dataset of 4,682 craters (min diameter 6 px) across 8 globally-distributed lunar regions, converted to 1,604 labeled 640×640 image blocks.
- WHY RELEVANT: A trained, illumination-robust crater detector is a reusable building block for *any* crater-based correspondence or geomorphological-feature approach we might adopt — not limited to CNSFM specifically.
- HOW IT COULD INTEGRATE: Could serve as the front-end feature/landmark detector for our own correspondence pipeline (whether we implement full CNSFM or a different matching strategy on top of detected craters).
- WHAT WOULD NEED TO CHANGE: Trained specifically on LROC NAC imagery at 0.5 m/pixel; would need retraining or domain adaptation for OHRC (finer resolution, ~0.3 m per Paper 2) and definitely for IIRS (80 m, likely too coarse for individual crater detection) and TMC (untested by any paper so far).
- ADVANTAGE: [REPORTED] "Recall and precision are close to 1" on validation set; empirically shown resistant to illumination changes (Fig. 9 detection consistency across large incidence/azimuth differences).
- CHALLENGE: Manual annotation effort (authors deliberately annotated only "easily identifiable, clear and preferably circular" craters) — bias toward well-defined craters may reduce detector utility in terrain with degraded/irregular craters.
- OPPORTUNITY: Could combine with our own dataset from OHRC/TMC/IIRS to build a multi-sensor crater detector via further transfer learning.

### Mismatched CNSF Removal (MCR) — local geometric transformation consistency check
- WHAT: An outlier-rejection stage separate from RANSAC: verifies each candidate CNSF match by checking whether a local similarity transform derived from it is consistent with a sufficient number of other candidate matches (Eq. 11).
- WHY RELEVANT: Our problem statement explicitly requires "outlier rejection" — this is a structurally different (non-RANSAC) approach worth comparing/combining with the RANSAC approach used in Paper 2.
- HOW IT COULD INTEGRATE: Could be used as a pre-filter before RANSAC, or as an alternative outlier-rejection mechanism specifically for structure/topology-based matches (as opposed to point-descriptor matches).
- WHAT WOULD NEED TO CHANGE: Method is specifically designed for CNSF-type structural matches; would need adaptation for general point-feature outputs (e.g., from SuperGlue).
- ADVANTAGE: [DEMONSTRATED] Ablation study (Table 6) shows MCR alone raises average RCM from 62.2% to 100% on Scene S1 — a very large, directly measured contribution.
- CHALLENGE: Relies on having enough "correct" CNSF pairs already present to establish a valid consensus transformation; may struggle in extremely sparse-match scenarios.
- OPPORTUNITY: Directly quantifies the value of structure-consistency-based outlier rejection — worth adopting as a design pattern regardless of which front-end matcher we choose.

---

## 4. P2 — RESEARCH THEORY / TECHNICAL INSIGHT

### CONCEPT: Geometric/topological features vs. pixel/gradient features under nonlinear illumination
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED] SIFT ("most sensitive to illumination changes") and even illumination-adapted spatial/frequency-domain methods (HAPCG, ML-HLMO — both explicitly designed for radiometric robustness) still degrade substantially under extreme Sun-angle differences, while a purely geometric/topological representation (crater center + diameter only, no pixel appearance at all) remains stable.
- WHY IT MATTERS: Directly informs an architectural choice for our project: illumination robustness can be pursued either by *making descriptors more robust to intensity change* (RIFT2, HAPCG, ML-HLMO, WSSF — the approach of most existing methods including Paper 2's candidates) or by *eliminating dependence on pixel intensity altogether* (this paper's approach). The evidence here favors the latter as more robust in the most extreme conditions.
- PRACTICAL IMPLICATION: For OHRC/TMC (panchromatic, crater-visible) imagery, a geometric-landmark-based approach may outperform any descriptor-robustness approach in extreme illumination; for IIRS (hyperspectral, coarser resolution) this approach is unlikely to be directly applicable due to insufficient spatial resolution for reliable crater detection.

### CONCEPT: Failure regime is not "Sun-angle magnitude" alone but shadow-driven overlap loss
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED/REPORTED] Discussion section attributes remaining CNSFM failures in Scene S3 (near south pole) not simply to large incidence-angle values, but specifically to cases where near-180° azimuth differences cause the *shared/overlapping terrain* between the two images to become predominantly shadow-covered on at least one side, leaving too few visible/detectable craters in the common region.
- WHY IT MATTERS: Refines our understanding of "illumination robustness" — it's not just about tolerating different Sun angles per se, but specifically about surviving cases where different parts of the same terrain are illuminated vs. shadowed between the two images (a genuinely harder, more specific problem than uniform brightness/contrast shift).
- PRACTICAL IMPLICATION: Evaluation and solution design should specifically stress-test large-azimuth-difference pairs (not just large-incidence-angle pairs) since these produce a qualitatively distinct failure mode (loss of overlapping visible/matchable terrain, not just appearance distortion).

### CONCEPT: Sparse/young terrain (few craters) as an inherent limitation of any crater-dependent approach
- WHAT THE PAPER ESTABLISHES: [REPORTED] Authors explicitly acknowledge that "in some geologically young units (e.g., light plains or impact melt ponds), craters might be relatively sparse, which can also result in matching failure" and that "these are limitations common to structure-dependent crater feature matching algorithms."
- WHY IT MATTERS: A crater-geometry-only approach (CNSFM or similar) cannot be a complete, universal solution for our project — certain terrain types will always lack the density of landmarks it needs.
- PRACTICAL IMPLICATION: [INFERENCE] Supports a hybrid-method design: crater-geometry matching for crater-rich terrain, falling back to a descriptor-based or deep-learning matcher (e.g., SuperGlue from Paper 2) for crater-sparse regions such as maria/melt ponds.

---

## 5. P3 — EXISTING / PAST SOLUTIONS

| APPROACH | PROBLEM ADDRESSED | CORE METHOD | RESULT | STRENGTH | LIMITATION | RELEVANCE TO US |
|---|---|---|---|---|---|---|
| SIFT (baseline) | General feature matching | Gradient-based descriptor, DoG scale-space | [DEMONSTRATED] Lowest SR across all 3 scenes (33.3%, 24.4%, 17.3%); "most sensitive to illumination changes" | Simple, fast, widely available | Fails under nonlinear radiometric distortion caused by lunar shading/shadowing | High — confirms Paper 2's finding independently on a different dataset (LROC NAC vs OHRC/IIRS) |
| HAPCG [frequency-domain phase congruency method] | Multi-modal/illumination-robust matching via frequency domain | Phase congruency-based feature description | [DEMONSTRATED] Moderate SR (66.7%, 48.9%, 22.1%); "some resistance but unstable, prone to mismatches" | Better than SIFT under illumination variation | Still fails significantly in polar/high-azimuth-difference regions | Medium — represents the "robust descriptor" strategy family, useful as comparison baseline |
| ML-HLMO (Multiscale Histogram of Local Main Orientation) | Radiometric-invariant remote sensing image registration | Histogram-of-orientation-based multi-modal descriptor | [DEMONSTRATED] Inconsistent — very high on S2 equatorial (88.9%) but poor at pole (13.9%) — least robust to azimuth differences (34° max tolerated) | Can perform very well under certain illumination regimes | Highly inconsistent/unstable across regions; large numbers of matches but lower reliability (RCM as low as 52% in S3) | Medium — cautionary example of a method that looks strong in easy conditions but fails to generalize |
| WSSF (Weighted Structure Saliency Feature) | Multi-modal matching using structural saliency | Structure-based (not purely pixel-gradient) descriptor | [DEMONSTRATED] Second-best overall — 100%/66.7%/31.2% SR, best RMSE/RCM among the four baselines, tolerates up to 62° azimuth difference | Best-performing non-proposed baseline; benefits from partial use of structural information | Still substantially worse than CNSFM at the pole (31.2% vs 72.3%) | High — closest existing competitor to CNSFM; confirms that structural (vs purely intensity-based) information is the right general direction |
| CNSFM (this paper's method) | Illumination-robust lunar correspondence via crater geometry | Deep-learning crater detection + similarity-invariant geometric topology matching + dedicated outlier removal | [DEMONSTRATED] Best across all metrics/scenes: 100/100/72.3% SR, avg RMSE 1.0/1.5/2.2 px, RCM 100/99.3/100%, tolerates up to 147.7° azimuth difference | Most robust to illumination/azimuth variation of all tested methods; near-perfect reliability (RCM) | Fails in crater-sparse terrain; fails when overlap region is almost entirely shadow-covered; NCM (raw match count) is relatively low (tens, not hundreds) due to reliance on limited crater counts | Very High — current best-in-class illumination-invariant method demonstrated on real lunar orbiter data |

---

## 6. P4 — WHAT NOT TO TRY / FAILURE MODES

### METHOD: SIFT (and by extension any purely gradient/intensity-based descriptor) for high-Sun-angle-difference lunar pairs
- WHY PEOPLE USE IT: Simplicity, speed, ubiquity, no training data required.
- FAILURE / WEAKNESS: [DEMONSTRATED] Lowest success rate of all five tested methods in every one of the three test regions (33.3%, 24.4%, 17.3%), and explicitly identified as "the most sensitive to illumination changes."
- WHY IT FAILS: Gradient-orientation-based descriptors change substantially when shadow boundaries shift with Sun azimuth/incidence angle, since shadows create strong artificial edges/gradients unrelated to the true terrain structure.
- CONDITIONS WHERE IT FAILS: Any lunar image pair with large incidence-angle or azimuth-angle differences; systematically worse near the poles.
- IMPLICATION FOR OUR PROJECT: Third independent confirmation (after Paper 1's cited literature and Paper 2's direct experiments) that plain SIFT is inadequate for illumination-variant lunar correspondence — should not be relied upon as primary matcher for our polar/high-Sun-angle-difference cases.
- POSSIBLE ALTERNATIVE: CNSFM-style geometric matching (best performer here) or SuperGlue (best performer in Paper 2) — see P5 for combining both.

### METHOD: "Illumination-robust" descriptor methods (HAPCG, ML-HLMO) that are robust in principle but empirically unstable
- WHY PEOPLE USE IT: Designed explicitly to handle nonlinear radiometric distortion via frequency-domain or histogram-of-orientation approaches — theoretically well-suited to our illumination-robustness requirement.
- FAILURE / WEAKNESS: [DEMONSTRATED] Despite being purpose-built for illumination robustness, both showed large performance swings — e.g., ML-HLMO ranged from 88.9% SR (S2) down to 13.9% SR (S3), and both are described as "unstable and prone to mismatches" even where they succeed.
- WHY IT FAILS: [INFERENCE] Being theoretically illumination-invariant in their descriptor formulation does not guarantee empirical stability under the specific compound conditions of lunar polar imagery (extreme shadow + weak texture + rugged terrain simultaneously).
- CONDITIONS WHERE IT FAILS: Polar/high-azimuth-difference cases specifically; performance is inconsistent even within "moderate" illumination-difference conditions.
- IMPLICATION FOR OUR PROJECT: Don't assume a method labeled "illumination-robust" or "multi-modal" in its own literature will generalize reliably to the polar extremes relevant to our problem statement — empirical validation on our actual target data/regions is essential, not just method category.
- POSSIBLE ALTERNATIVE: Structure/topology-based methods (WSSF, CNSFM) showed more consistent — though not perfect — robustness in this specific benchmark.

### METHOD: Any single crater-geometry-only approach as a universal/standalone solution
- WHY PEOPLE USE IT: Crater geometry is illumination-invariant by construction and (per this paper) the most robust option demonstrated so far.
- FAILURE / WEAKNESS: [REPORTED] Explicitly fails in (a) crater-sparse terrain (young geological units, mare, impact melt ponds) and (b) cases of near-total shadow coverage of the overlap region (Scene S3 failure cases, Figure 17).
- WHY IT FAILS: The method's entire matching unit depends on having enough detectable craters within the shared/overlapping field of view; if craters are absent or the shared region is shadow-obscured, there's no fallback signal.
- CONDITIONS WHERE IT FAILS: Crater-sparse geological units; near-180° azimuth differences combined with limited image overlap.
- IMPLICATION FOR OUR PROJECT: Should not adopt CNSFM (or similar) as a sole/universal solution — needs to be paired with a complementary matcher (e.g., a descriptor/deep-learning method) for crater-sparse regions, consistent with our problem statement's call for a "generic" solution across varied lunar terrain.
- POSSIBLE ALTERNATIVE: Hybrid approach — CNSFM (or similar geometric-landmark method) as primary for crater-rich regions, SuperGlue-style deep matcher as fallback/complement for crater-sparse regions (see P5).

---

## 9. SOLUTION / DECISION MATRIX

| Finding / Method | What it solves | Why useful | Why NOT use it | Challenge | Opportunity | Relevance |
|---|---|---|---|---|---|---|
| CNSFM (crater geometry + similarity-invariant matching) | Illumination/Sun-angle-invariant correspondence | Best-in-class robustness to extreme Sun-angle/azimuth variation; near-perfect reliability (RCM) | Fails in crater-sparse terrain; fails under near-total shadow overlap loss | Needs sensor-specific crater detector (OHRC/TMC untested) | Directly reusable code+dataset; adapt to our sensors | P0 |
| Mismatched CNSF Removal (MCR) | Outlier rejection for structural matches | Demonstrated massive RCM improvement (62.2%→100%) | Specific to structural/topological match types | Adapting to point-descriptor outputs | Combine with RANSAC as a structure-consistency layer | P1 |
| YOLOv9 crater detector (transfer-learned) | Illumination-robust landmark detection | High recall/precision even under illumination change | Trained only on LROC NAC @ 0.5m; needs retraining for our sensors | New annotated dataset needed for OHRC/TMC | Reusable detection backbone across projects | P1 |
| SIFT / HAPCG / ML-HLMO (baselines) | General/illumination-aware matching | Reference comparison points | All substantially underperform CNSFM, esp. at poles | — | Use only as baseline/comparison, not primary method | P3/P4 |
| WSSF | Structural-saliency-based multi-modal matching | Best non-proposed baseline; partial structural robustness | Still well below CNSFM in extreme conditions | — | Possible secondary/ensemble candidate | P3 |
| Hybrid CNSFM + SuperGlue | Full requirement coverage across terrain types & challenge types | Combines best-demonstrated illumination robustness (CNSFM) with best-demonstrated general/multi-modal robustness (SuperGlue, Paper 2) | Untested combination; added engineering complexity | Terrain-adaptive switching logic; result reconciliation | Most complete solution space explored so far across all 3 papers | P5 |

---

## 10. KEY TAKEAWAYS FOR OUR PROJECT
1. [DEMONSTRATED] CNSFM is currently the strongest illumination-invariant method reviewed across all three papers — 72.3% success rate at the lunar south pole vs. a best of 31.2% among four other tested methods (including SIFT), directly validating a geometry-over-intensity strategy for our project's #1 stated challenge.
2. A third independent source (after Paper 1's literature review and Paper 2's direct experiments) confirms SIFT's fundamental inadequacy under Sun-angle/illumination variation — this is now a very well-established finding across all reviewed literature and should not be re-litigated; treat classical intensity/gradient descriptors as a baseline only.
3. CNSFM has a clear, paper-acknowledged limitation: crater-sparse terrain (young geological units, maria, melt ponds) — our "generic" solution requirement means CNSFM alone cannot be the complete answer; a hybrid approach (crater-geometry + a general descriptor/deep-learning matcher like SuperGlue from Paper 2) is a promising, evidence-backed direction.
4. New, more precise understanding of the illumination-failure mechanism: failures are driven specifically by near-180° solar-azimuth differences causing shadow-driven loss of *shared visible terrain* between image pairs — not simply "high incidence angle" — this should directly inform how we construct our own evaluation/stress-test dataset.
5. Directly reusable resources exist: public code + dataset (https://github.com/Bin501/CNSFM) and a public training methodology (YOLOv9 transfer learning, 4,682 annotated craters) — significantly lowers the barrier to prototyping this approach for our project.
6. TMC (one of our three named primary sensors) remains untested by all three papers reviewed so far — Paper 1 used TMC only for radiometric mosaicking, Paper 2 tested OHRC/IIRS/DFSAR but not TMC, and this paper (Paper 3) uses only LROC NAC. This is now a confirmed, persistent research gap across our entire literature base.
7. Outlier rejection via structural/geometric transformation consistency (MCR) is a validated, high-impact technique (RCM 62.2%→100% in ablation) — worth incorporating into our pipeline design regardless of which front-end matcher(s) we ultimately choose.
8. Sub-pixel accuracy requirement: CNSFM's average RMSE (1.0–2.2 px depending on region) is in a similar range to Paper 2's SuperGlue results (~0.4–0.9 px) but generally slightly higher — suggesting SuperGlue may still have an edge on raw geometric accuracy where it succeeds, while CNSFM has the edge on success rate/robustness in the hardest illumination cases. This trade-off is directly relevant to our sub-pixel accuracy requirement and supports further investigating a combined approach.

---
