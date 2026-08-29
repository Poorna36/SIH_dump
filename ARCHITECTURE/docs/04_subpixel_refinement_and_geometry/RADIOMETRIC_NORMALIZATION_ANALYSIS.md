# Deep Radiometric Normalization for Cross-Sensor Lunar Mosaics
(cGAN, TMC+SELENE → WAC reference)

Reference to SIH 26166: Multi-modal, Sun-angle and scale-invariant image correspondence (OHRC/TMC/IIRS)

---

## 1. RELEVANCE SNAPSHOT
- Overall relevance: Low–Medium
- Most relevant component: cGAN-based nonlinear radiometric/illumination normalization across sensors
- Why it matters to OUR project: Our problem is correspondence/registration under Sun-angle variation, not mosaic seam-blending. Illumination normalization could serve only as a *preprocessing* step to make matching easier.
- Most valuable takeaway: [DEMONSTRATED] The paper explicitly excludes geometric alignment — it assumes images are already registered/mosaicked. This is the inverse of our core problem, so it is a candidate preprocessing tool at best, not a solution.

---

## 2. P0 — DIRECT CONNECTION / USE NOW
None.
[DEMONSTRATED] The method operates on already-mosaicked (i.e., already geometrically co-registered) imagery. It performs no feature detection, correspondence, or geometric registration. The Discussion section explicitly states the framework "does not address geometric misalignment or parallax effects."

---

## 3. P1 — HIGHLY RELEVANT / POTENTIALLY INTEGRABLE

### cGAN Radiometric Normalization (U-Net generator + PatchGAN discriminator)
- WHAT: Learns a nonlinear pixel-level mapping from an illumination-inconsistent multi-sensor input (TMC + SELENE mosaic) to a photometrically stable reference (LROC WAC), via adversarial + L1 loss.
- WHY RELEVANT: Sun-angle/illumination variation and multi-sensor appearance differences are two of our four core challenges. A pre-normalization step could reduce appearance gap before feature/keypoint matching.
- HOW IT COULD INTEGRATE: Use as a *preprocessing module* — normalize source and reference patches toward a shared radiometric domain before running correspondence detection (SIFT/deep features/etc.), rather than as the correspondence method itself.
- WHAT WOULD NEED TO CHANGE:
  - Needs paired training data (distorted patch ↔ clean reference patch) — not confirmed available for OHRC/TMC/IIRS ↔ LRO NAC/SELENE pairs at matching resolution.
  - Trained per fixed mosaic-scale domain (512×512 / 1024×1024 patches on full mosaics); would need re-purposing for smaller, arbitrary image-pair patches used in feature matching pipelines.
  - No guarantee that normalization preserves the fine structural cues (crater rims, small-scale texture) needed for sub-pixel keypoint accuracy — only validated via SSIM/PSNR against a WAC reference, not against keypoint repeatability.
- ADVANTAGE: [DEMONSTRATED] Learned mapping outperforms classical histogram/Retinex normalization in tonal consistency (higher SSIM, reduced seam artifacts); adapts automatically without manual per-image tuning.
- CHALLENGE: [REPORTED] Requires a "high-quality radiometric reference dataset" (author-acknowledged limitation) — may not exist for OHRC (5m+ resolution) vs LRO NAC at comparable scale; GAN training instability requires careful tuning.
- OPPORTUNITY: [HYPOTHESIS] Could be adapted into a lightweight "illumination-normalization front-end" feeding into a downstream illumination-invariant descriptor/matcher, rather than being the end product.

---

## 4. P2 — RESEARCH THEORY / TECHNICAL INSIGHT

### CONCEPT: Limits of classical normalization for nonlinear illumination variation
- WHAT THE PAPER ESTABLISHES: [REPORTED] Histogram matching and Retinex-based methods are computationally efficient but "limited in handling nonlinear variations caused by changes in illumination geometry and sensor response."
- WHY IT MATTERS: Confirms that simple intensity-based normalization (a common first idea for illumination robustness) is insufficient for lunar Sun-angle variation — directly relevant since our problem explicitly flags illumination robustness as a hard requirement.
- PRACTICAL IMPLICATION: Don't rely solely on histogram equalization/CLAHE as the illumination-robustness strategy; it should at most be a light preprocessing step, not the core solution (see P4).

### CONCEPT: Patch-based processing with overlap-aware blending for large planetary imagery
- WHAT THE PAPER ESTABLISHES: [DEMONSTRATED] Fixed-size patches (512×512, 1024×1024) with overlap-aware weighted-average reconstruction handle large mosaics efficiently while reducing boundary artifacts.
- WHY IT MATTERS: Generic engineering pattern relevant to any deep-learning pipeline (e.g., a learned feature/descriptor network) applied to large OHRC/TMC scenes that exceed typical network input size.
- PRACTICAL IMPLICATION: [INFERENCE] If we deploy a CNN-based descriptor/detector, patch-tiling + overlap blending is a reusable strategy for full-scene inference.

---

## 5. P3 — EXISTING / PAST SOLUTIONS

| APPROACH | PROBLEM ADDRESSED | CORE METHOD | RESULT | STRENGTH | LIMITATION | RELEVANCE TO US |
|---|---|---|---|---|---|---|
| Wang et al. / Li et al. (Chang'E-1/2 mosaicking) | Global lunar mosaic generation | Classical geometric alignment + tie-point extraction, global radiometric balancing | [REPORTED] Reliable under controlled, single-mission conditions | Automated pipeline | Poor multi-mission/radiometric adaptability | Low — geometric alignment only within single sensor, not cross-sensor correspondence |
| SIFT (Lowe) / Laplacian feature matching (Zhang et al.) | Feature-based lunar image registration | Scale-invariant keypoints / Laplacian matching | [REPORTED] Strong geometric alignment performance | Established, well-understood | No radiometric correction; visible seams under illumination variation | Medium — SIFT is a baseline correspondence method directly relevant to our task, but paper confirms its weakness under illumination changes (see P4) |
| Histogram matching / Multiscale Retinex (Jobson, Rahman) | Intensity/tonal standardization | Statistical/perceptual normalization | [REPORTED] Computationally efficient, limited generalization | Fast, no training | Fails under nonlinear illumination geometry changes; over-enhancement artifacts | Medium — relevant as a *baseline to avoid over-relying on* for illumination robustness |
| Photometric normalization via LROC WAC reference (Sato et al.) | Physically-grounded radiometric correction | Calibrated reference-based normalization | [REPORTED] Improved global consistency | Physically grounded | Requires carefully calibrated reference; not adaptable to heterogeneous multi-source data | Low-Medium — reference-based normalization concept could inform preprocessing design |
| Proposed cGAN (this paper) | Radiometric/tonal seam removal across TMC+SELENE→WAC | U-Net generator + PatchGAN discriminator, adversarial + L1 loss | [DEMONSTRATED] Best PSNR 30.55 dB, SSIM 0.9876 at epoch 125 | Learns nonlinear, spatially-adaptive mapping automatically | Needs paired reference data; no geometric correction; computational cost | Low-Medium — potential preprocessing only |

---

## 6. P4 — WHAT NOT TO TRY / FAILURE MODES

### METHOD: Classical histogram matching / global tone balancing for illumination robustness
- WHY PEOPLE USE IT: Computationally cheap, no training data or model needed, simple to implement.
- FAILURE / WEAKNESS: [REPORTED] Cannot model nonlinear radiometric variation caused by differing Sun angles and sensor response.
- WHY IT FAILS: Global/block-wise intensity adjustment assumes a roughly uniform or linear shift in brightness; actual lunar illumination changes (shadow growth/shrinkage, terrain-dependent reflectance) are spatially nonuniform and nonlinear.
- CONDITIONS WHERE IT FAILS: Cross-sensor, cross-illumination-angle pairs — exactly our project's core scenario.
- IMPLICATION FOR OUR PROJECT: Do not rely on histogram equalization/CLAHE alone as an illumination-invariance strategy for feature matching; expect it to leave residual appearance gaps that break intensity/gradient-based matchers.
- POSSIBLE ALTERNATIVE: Illumination-invariant feature descriptors (not covered in this paper) or a learned normalization step (as above) used only as light preprocessing, combined with descriptors robust to residual differences.

### METHOD: SIFT / Laplacian-based feature matching alone (as cited, not this paper's method)
- WHY PEOPLE USE IT: Strong, well-established general-purpose feature matching with scale/rotation invariance.
- FAILURE / WEAKNESS: [REPORTED, cited from related work] "Primarily address geometric consistency and do not explicitly account for radiometric discrepancies," producing visible seams/tonal discontinuities under varying illumination.
- WHY IT FAILS: SIFT-type gradient-based keypoints are sensitive to intensity/gradient pattern changes caused by shifting shadows under different Sun angles — a known weakness for planetary imagery.
- CONDITIONS WHERE IT FAILS: Large Sun-angle differences between source/reference lunar images.
- IMPLICATION FOR OUR PROJECT: Confirms (secondhand, via this paper's literature review) that plain SIFT is likely insufficient as a standalone solution for our illumination-variant requirement — consistent with expectations already in our problem statement.
- POSSIBLE ALTERNATIVE: Illumination-robust/learned descriptors, or combine SIFT with a radiometric-normalization preprocessing stage (per P1 above).

---

## 9. SOLUTION / DECISION MATRIX

| Finding / Method | What it solves | Why useful | Why NOT use it | Challenge | Opportunity | Relevance |
|---|---|---|---|---|---|---|
| cGAN radiometric normalization (U-Net+PatchGAN) | Tonal/seam inconsistency in mosaics | Learns nonlinear illumination mapping automatically | Not a correspondence/registration method; assumes pre-registered images; needs paired data | Requires calibrated reference dataset at matching resolution | Possible preprocessing step to reduce illumination gap before matching | P1 |
| Histogram/Retinex normalization (cited baseline) | Cheap tonal standardization | No training needed | Fails under nonlinear illumination/sensor differences | — | Use only as light preprocessing, not primary illumination-robustness strategy | P4 |
| SIFT/Laplacian matching (cited baseline) | Geometric feature correspondence | Established, scale/rotation invariant | No radiometric robustness — degrades under Sun-angle variation | Combine with normalization | Use as one component in hybrid pipeline, not standalone | P3/P4 |
| Patch-based + overlap-aware inference | Scalable processing of large imagery | Reusable engineering pattern | N/A | Boundary artifact handling | Reusable for any CNN-based descriptor/detector on large OHRC/TMC scenes | P2 |

---

## 10. KEY TAKEAWAYS FOR OUR PROJECT
1. This paper does not solve our problem — it performs radiometric/tonal normalization on already-registered mosaics, not feature correspondence or geometric registration. Explicitly out-of-scope per authors.
2. [HYPOTHESIS] The cGAN normalization concept *could* be adapted as an illumination-normalization preprocessing step before correspondence matching — but this is speculative and unvalidated for our use case (sub-pixel accuracy, small patches, cross-sensor).
3. Confirms (via cited literature) that classical histogram/Retinex normalization is insufficient for nonlinear Sun-angle illumination changes — avoid relying on it alone for illumination robustness.
4. Confirms (via cited literature) that plain SIFT/Laplacian matching lacks radiometric robustness — supports our problem statement's premise that ordinary gradient-based matching struggles under illumination variation.
5. Patch-based tiling + overlap-aware blending is a reusable engineering pattern if we deploy any CNN-based component on full-scene OHRC/TMC imagery.
6. Major feasibility gap: any GAN-based normalization approach requires a paired, calibrated radiometric reference dataset — availability at OHRC/TMC/IIRS-to-LRO-NAC resolution is unconfirmed and a likely bottleneck.
7. Research gap highlighted (paper's own future work): authors themselves flag "physics-informed learning strategies incorporating photometric models of lunar illumination" as unexplored — potentially a more directly relevant future direction than this paper's cGAN approach.
8. Overall recommendation: treat this paper as a P1/P2 supporting reference only — useful for confirming known weaknesses of classical methods, not for direct architectural reuse.

---
