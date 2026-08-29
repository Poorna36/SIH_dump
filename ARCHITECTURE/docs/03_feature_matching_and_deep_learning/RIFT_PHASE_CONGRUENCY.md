# RIFT: Radiation-invariant Feature Transform (Li, Hu, Ai)

## 1. RELEVANCE SNAPSHOT
- Overall relevance: Very High
- Most relevant component: Phase-congruency (PC) detection + Maximum Index Map (MIM) descriptor for NRD/multi-sensor robustness
- Most valuable takeaway: RIFT [DEMONSTRATED] hits 100% success rate across 6 NRD datasets (SIFT: 31.7%, SAR-SIFT: 28.3%), ME ~1.8px / RMSE ~1.9px — but is explicitly not scale-invariant, a direct gap vs. our scale requirement.

## 2. P0 — DIRECT CONNECTION / USE NOW

### Phase Congruency (PC) Detection
- WHAT: Log-Gabor filter bank (scales Ns, orientations No) → PC map → minimum moment map (corners) + maximum moment map (edges, via FAST).
- WHY IT CONNECTS: PC is illumination/contrast-invariant by construction — directly targets our Sun-angle robustness requirement.
- USE: Corner+edge combined detection improves both repeatability and match count → supports our "uniform distribution" requirement.
- Evidence: [DEMONSTRATED] Fig.1 — PC-moment detection stable under NRD where intensity-based FAST fails.
- Implementation note: Needs param tuning per dataset (paper used No=6, Ns=4, J=96 on map-optical); would need re-tuning for lunar imagery.

### Maximum Index Map (MIM) Descriptor
- WHAT: Argmax-orientation-channel index per pixel (from same log-Gabor convolution, cheap reuse) → 6×6×No histogram, illumination-normalized.
- WHY IT CONNECTS: Core solution for cross-sensor matching (OHRC↔TMC↔IIRS↔LRO-NAC) where intensity/gradient correlation fails.
- Evidence: [DEMONSTRATED] Note — PC map alone as descriptor failed (near-all outliers, Fig.2d); MIM was needed. Important negative result (see P4).
- Implementation note: Rotation invariance needs No candidate MIMs per keypoint at match time — added compute.

### Rotation Invariance (Multi-MIM Reordering)
- WHAT: Dominant-orientation normalization doesn't work for MIM; instead build No candidate MIMs (different starting layers) per target keypoint, match against all.
- Evidence: [DEMONSTRATED] Fig.10 — full 360° rotation test (72 pairs, 5° steps), 100% success, NCM always >40.
- Implementation note: Multiplies descriptor cost by No(=6) for target image.

## 3. P1 — HIGHLY RELEVANT / INTEGRABLE

### Adding Scale Invariance to RIFT
- WHY RELEVANT: RIFT has no scale invariance; paper pre-resamples to matching GSD. Our sensors (OHRC/TMC/IIRS/LRO-NAC) differ in resolution.
- INTEGRATION: Add multi-octave log-Gabor scale-space search (SIFT-style) on top of RIFT's PC/MIM core.
- CHALLENGE: Compute cost — already high (Ns × No × rotation candidates); scale-space multiplies further.
- OPPORTUNITY: This gap is a plausible genuine research contribution for our project.

### Outlier Rejection (NBCS)
- WHY RELEVANT: Matches our "reliable outlier rejection" requirement.
- INTEGRATION: Swap in NBCS (authors' own prior method, not detailed here) or standard RANSAC/affine-consensus.
- CHALLENGE: NBCS specifics require the cited paper [3].
- OPPORTUNITY: Modular — can benchmark multiple back-ends against inlier-ratio/RMSE.

## 4. P2 — RESEARCH THEORY

CONCEPT: Phase vs. Intensity/Gradient Invariance
- ESTABLISHES: [DEMONSTRATED] PC is highly invariant to contrast/illumination/scale (frequency-domain phase alignment, not pixel amplitude).
- IMPLICATION: Confirms theoretical basis for why Sun-angle/shadow changes should not corrupt PC features the way they corrupt SIFT gradients.

CONCEPT: Detector Robustness ≠ Descriptor Robustness
- ESTABLISHES: [DEMONSTRATED] PC map detects well but describes poorly (low info content, noise-sensitive, mostly edges).
- IMPLICATION: Must evaluate detection stability and description discriminability separately for any lunar method.

## 5. P3 — EXISTING / PAST SOLUTIONS

| APPROACH | METHOD | RESULT | LIMITATION | RELEVANCE |
|---|---|---|---|---|
| SIFT | Gaussian scale-space + gradient histogram | [DEMONSTRATED] 31.7% avg SR; 0% on depth-optical | Gradient breaks under NRD | Baseline to beat; unsuitable alone for cross-sensor lunar matching |
| SAR-SIFT | Redefined gradient + multiscale Harris | [DEMONSTRATED] 28.3% avg SR; 0 keypoints on some images; "extremely unstable" | Redefined gradient even more NRD-sensitive; Harris fails on weak structure | Confirms Harris/gradient risk for low-texture lunar terrain |
| HOPC | Harris + orientation-PC + template matching (HOPCncc), needs geo-metadata pre-registration | [REPORTED] Effective if geometry known | (1) "Completely unusable" without accurate geolocation (2) rotation/scale-sensitive beyond small search window (3) Harris fails badly on depth maps | Caution: our TBD dataset metadata may be unreliable — avoid metadata-dependent designs |
| UR-SIFT | Entropy-based feature selection for uniform distribution | Not tested here | Still gradient-based | P5 candidate: graft uniformity strategy onto PC/MIM pipeline |

## 6. P4 — WHAT NOT TO TRY

Intensity/gradient detectors (SIFT, Harris, FAST-on-raw, SAR-SIFT's gradient)
- FAILS BECAUSE: Amplitude-based; NRD/cross-sensor differences distort or invert pixel values (e.g., optical vs. infrared contrast reversal).
- CONDITIONS: Cross-sensor pairs, low-texture surfaces (depth maps — analogous to smooth lunar terrain).
- IMPLICATION: OHRC/TMC/IIRS vs. LRO-NAC will have sensor + Sun-angle differences — plain SIFT/ORB/Harris is a demonstrated failure mode, not just a theoretical risk.
- ALTERNATIVE: PC/log-Gabor phase-domain detection (RIFT-style).

PC map used directly as descriptor
- FAILS BECAUSE: Low information (near-zero values), noise-sensitive, edge-dominated → near-all-outlier matches despite good detection (Fig.2d).
- IMPLICATION: If building a custom PC pipeline, don't stop at PC map for description — need MIM-like richer encoding.

HOPC-style geo-metadata-dependent pre-registration
- FAILS BECAUSE: Entire method depends on accurate a-priori geometric correction; satellite geolocation can be off by "hundreds of meters."
- IMPLICATION: Since our dataset/metadata is TBD, avoid designs requiring accurate a-priori geo-registration.
- ALTERNATIVE: Metadata-independent methods like RIFT.

## 9. SOLUTION / DECISION MATRIX

| Method | Solves | Why Useful | Why NOT (as-is) | Challenge | Opportunity | Relevance |
|---|---|---|---|---|---|---|
| PC detection | Illumination-robust keypoints | Matches Sun-angle requirement | Needs scale extension | Compute cost | Core detector | P0 |
| MIM descriptor | NRD-robust cross-sensor description | Cheap, proven on 6 modalities | Not scale-invariant | Scale-space wrapper needed | Core descriptor | P0 |
| Multi-MIM rotation invariance | Viewpoint rotation | 100% SR over 360° | Adds No× compute | Match-time efficiency | Reuse as-is | P0 |
| HOPC | Multi-modal template matching | Best-paper method | Needs geo-metadata (uncertain for us) | Metadata availability | Maybe for local refinement only | P3/P4 |
| Scale-space RIFT extension | Closes scale gap | Directly fills our requirement | Not demonstrated — our own contribution | Compute, novel engineering | Potential differentiator | P5 |
| UR-SIFT uniform sampling | Match distribution | Directly addresses SIH requirement | Standalone still gradient-based | Integration | Low-risk add-on | P1/P5 |
| Plain SIFT/Harris | — | — | Fails under NRD/cross-sensor/low-texture | — | — | P4 — avoid |

## 10. KEY TAKEAWAYS
1. Use now: RIFT's PC+MIM core is a strong evidence-backed candidate for Sun-angle/multi-sensor robustness (100% SR vs. ~30% for SIFT/SAR-SIFT).
2. Investigate: RIFT lacks scale invariance — extending with multi-scale log-Gabor search is a genuine open gap matching our exact requirement.
3. Avoid: Gradient/intensity detectors (SIFT, Harris, FAST, SAR-SIFT) as core method — demonstrated failure on cross-modal/low-texture conditions.
4. Avoid: Architectures requiring accurate a-priori geo-metadata (HOPC-style) — risky given our TBD dataset.
5. Constraint: RIFT's raw RMSE (~1.8–2.0px) isn't sub-pixel yet — needs added RANSAC + least-squares refinement stage.
6. Add-on: UR-SIFT's entropy-based uniform sampling is a low-cost way to satisfy the "uniform distribution" requirement.
7. Compute check: Full RIFT (multi-scale × multi-orientation × rotation candidates) is heavier than SIFT — benchmark against OHRC's high resolution.
8. Metrics alignment: RIFT's evaluation (NCM, RMSE, ME, SR) closely matches SIH's suggested metrics — good template for our own evaluation protocol.
