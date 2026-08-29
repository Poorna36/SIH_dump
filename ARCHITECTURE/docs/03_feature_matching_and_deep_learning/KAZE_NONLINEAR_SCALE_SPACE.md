# Coarse-to-Fine Registration Using Improved KAZE (I-KAZE) + DE-Optimization

*Note: uploaded file was a truncated `.crdownload`; repaired and recovered all 10 pages (text + tables + figures). Conclusion/References section was cut off by the truncation — everything below is drawn from Intro → Methods → Experimental Results only.*

## 1. Relevance Snapshot
- Relevance: High. Directly addresses sub-pixel accuracy, RMSE-based evaluation, and cross-sensor optical registration — all core SIH 26166 requirements.
- Key takeaway: [DEMONSTRATED] PC-weighted KAZE + MI/DE fine refinement hits RMSE 0.57–0.82 px on 5 cross-sensor optical pairs, but [DEMONSTRATED] the same pipeline fails on an optical-vs-SAR pair (near-zero matches) — a real boundary condition for our multi-sensor case.

## 2. P0 — Direct Use

I-KAZE (Phase-Congruency-weighted KAZE), coarse matching
- Standard KAZE keypoints (Hessian response in nonlinear/Perona-Malik scale space) → each keypoint scored by Phase Congruency (PC, illumination-invariant) × Hessian response → keep top F = fixed % (x=0.4) of image size.
- Fixes KAZE's uncontrolled keypoint count/poor repeatability under intensity change; fixed budget also aids "uniform match distribution."
- [DEMONSTRATED] Beat SURF/FAST/BRISK/ORB/RIFT on NCM and MI, lowest RMSE, on all 5 datasets (Table 1). Runtime higher than SURF/FAST/BRISK/ORB but far below RIFT.
- Use: transferable illumination-robust keypoint-scoring layer, not KAZE-specific.

FSC (Fast Sample Consensus) outlier rejection — used pre-transform-estimation; [REPORTED] only, no ablation vs. RANSAC given. Verify separately.

MI + DE/best/1/bin fine registration (sub-pixel stage)
- θ (affine params) seeded from I-KAZE coarse estimate (not random) → DE mutation/crossover/selection maximizes MI(θ), cubic B-spline joint histogram, multi-resolution coarse→fine search.
- [DEMONSTRATED] Beat SPSA-based MI (prior standard) on MI & RMSE, all 5 datasets, statistically significant (paired t-test p=0.0007 MI / p=0.0012 RMSE), for only slightly more compute (~30s vs ~29s typical).
- Tuned params: N (iterations)=20 (MI saturates ~20 iters), S (mutation scale)=0.8 — tuned for this Earth-imagery dataset, will need re-tuning for lunar data.
- Use: reusable "coarse feature affine → seed → MI/DE fine refinement" template.

Evaluation protocol — 50 manually-selected, evenly-distributed tie points; match "correct" (NCM) if error < 1.5px; RMSE over NCM; MI as secondary signal. Matches SIH's RMSE/inlier metrics; reusable ground-truthing protocol (threshold needs revisiting for lunar sub-pixel targets).

## 3. P1 — Integrable

Multi-resolution pyramid MI optimization — progressively refines from coarse to fine levels, restricting search to neighborhood of prior level's optimum. Speeds convergence, avoids local optima. Risk (paper-stated): if a coarse pyramid level lacks structural information, optimizer can converge wrong/diverge — relevant to low-texture lunar maria. Opportunity: pair with terrain-aware ROI weighting (down-weight flat maria, prioritize craters/rays).

## 4. P2 — Theory/Insight

- Phase Congruency: measures local phase alignment across scales/orientations, not intensity — theoretically illumination-invariant. [DEMONSTRATED] improves keypoint repeatability under cross-sensor radiometric differences. Directly relevant to our Sun-angle problem, though untested on true shadow (near-zero information) regions.
- Mutual Information vs Cross-Correlation: MI tolerates nonlinear inter-sensor intensity relationships; [REPORTED, from cited prior work] MI > CC for multimodal registration. Reasonable default similarity metric for our fine-registration stage.

## 5. P3 — Existing/Past Solutions (baselines benchmarked)

| Method | Result | Limitation | Relevance |
|---|---|---|---|
| SURF | Correct matches all sets, more NCM/RMSE than I-KAZE-losing | Lower precision than I-KAZE | Standard comparator |
| FAST | Failed (0 matches) on datasets 2,3; fastest runtime | Poor repeatability cross-sensor | Shows corner-only detectors can totally fail |
| BRISK | Correct all sets, mid-tier | Underperforms I-KAZE | Baseline |
| ORB | Good on sets 1,4,5; weak on 2,3 | Inconsistent across datasets | Instability signal |
| RIFT (multimodal-specific) | Failed on dataset 3; very high runtime (up to 90s) | Failed despite multimodal design; heavy compute | Purpose-built ≠ reliable |
| SPSA-based MI (fine reg.) | Worse MI/RMSE than proposed DE, all sets, significant | Lower accuracy ceiling | Fine-reg baseline |

## 6. P4 — What Not to Try

FAST as sole coarse detector: [DEMONSTRATED] zero correct matches on 2/5 cross-sensor sets — corner response alone isn't robust to radiometric/resolution gaps. Implication: high-risk for OHRC/TMC/IIRS given their resolution/radiometry differences. Use I-KAZE-style illumination-robust detectors instead.

RIFT at scale: [DEMONSTRATED] failed entirely on one dataset despite being purpose-built for radiometric differences, and 15–30x slower than SURF/BRISK/ORB. Implication: "multimodal-by-design" ≠ automatically safe; validate on our actual sensor pairs before adopting.

Full I-KAZE+MI/DE pipeline under large modality gaps: [DEMONSTRATED] (Fig. 9) on optical (ALOS FBS) vs SAR (UAVSAR): only ~4 correspondences found, visible misalignment in checkerboard result. [INFERENCE] PC-invariance and MI's statistical-dependence assumption are calibrated for optical-optical gaps, not the much larger optical-vs-SAR appearance gap. Implication: IIRS (hyperspectral) vs OHRC/TMC (panchromatic) may sit closer to this "large gap" regime than a simple resolution difference — test this pairing early, don't assume generalization.

## 9. Decision Matrix

| Method | Solves | Why Useful | Why Not Use As-Is | Opportunity | Relevance |
|---|---|---|---|---|---|
| I-KAZE (PC-weighted) | Coarse matching, illumination robustness | Best NCM/MI/RMSE, optical-optical | Untested on lunar shadow extremes | Reusable illum.-invariant scoring layer | P0 |
| MI+DE/best/1/bin | Sub-pixel refinement | Significant gain vs SPSA | Untested on non-affine distortion | Reusable refinement backbone | P0 |
| FSC | Outlier rejection | Standard SIH requirement | No RANSAC comparison given | Benchmark vs RANSAC/MAGSAC | P1 |
| Multi-res MI search | Efficiency, avoids local optima | Faster convergence | Can diverge on low-texture levels | Pair with terrain-aware ROI weighting | P1 |
| FAST | Coarse baseline | Fast | 0 matches, 2/5 cross-sensor sets | — | P4 |
| RIFT | Multimodal descriptor | Built for radiometric gap | Failed 1 dataset, 15-30x slower | Investigate only if I-KAZE insufficient | P4 |
| Full pipeline | Optical-optical sub-pixel reg. | SOTA in-domain | Fails under large modality gap (opt-vs-SAR) | Test IIRS-vs-OHRC/TMC gap early | P0/P4 boundary |

## 10. Key Takeaways
1. Use now: I-KAZE + FSC + MI/DE(best/1/bin) is a validated coarse-to-fine template aligned with SIH's sub-pixel/RMSE/inlier requirements — good prototyping baseline.
2. Investigate: whether affine suffices for OHRC/TMC/IIRS↔LRO NAC, or a projective/higher-order model is needed for viewpoint variation.
3. Investigate: IIRS-vs-OHRC/TMC may behave like a large modality gap (paper's optical-vs-SAR failure mode) — test early.
4. Avoid: corner-only detectors (FAST) as sole coarse basis for cross-sensor matching — demonstrated total failure.
5. Avoid/verify first: purpose-built multimodal descriptors (RIFT) aren't automatically reliable — failed on one case, heavy runtime.
6. Constraint: DE/PC parameters (N=20, S=0.8, x=0.4%) are tuned for 1–30m Earth optical imagery — expect re-tuning for lunar data.
7. Research gap: no treatment of true low-texture (maria) or deep-shadow conditions in this paper.
8. Reusable methodology: manual evenly-distributed tie points + fixed pixel-error threshold (1.5px here) for NCM/RMSE — adapt for our LRO NAC-referenced benchmark.
