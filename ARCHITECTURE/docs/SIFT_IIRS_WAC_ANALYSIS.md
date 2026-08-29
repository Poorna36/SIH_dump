# SIFT-IIRS-WAC (extracted)

## 1. RELEVANCE SNAPSHOT
- Overall relevance: Very High
- Most relevant component: SIFT-based feature matching + RANSAC-homography + GCP generation pipeline for cross-sensor lunar image registration (Chandrayaan-2 IIRS ↔ LRO-WAC)
- Why it matters: Near-direct precedent — same spacecraft, same core problem (multi-sensor, illumination-variant lunar correspondence), uses SIFT + RANSAC + GCP georeferencing, reports sub-pixel-class accuracy, and explicitly analyzes failure vs. Sun-angle/latitude — exactly the axes our project must test.
- Most valuable takeaway: [DEMONSTRATED] SIFT + Lowe's ratio test + RANSAC homography registers IIRS (80 m/px) to LRO-WAC (100 m/px) with mean RMSE ~73 m (sub-pixel) across ~200 strips, but degrades sharply beyond ±55° latitude and with large solar-incidence mismatch — both directly relevant constraints for our sensors/geometry.

## 2. P0 — DIRECT CONNECTION / USE NOW

### Tiled SIFT + Lowe's Ratio + RANSAC Homography Pipeline
- WHAT: Patch-based SIFT matching between source (IIRS band) and reference (LRO-WAC), filtered by ratio test + RANSAC homography verification per tile.
- HOW: Image split into overlapping tiles (tiles <½ grid size discarded) → per-tile percentile normalization (2nd/98th clip, scale to [0,1]) → statistic-matching (mean/std transfer, Eq.2) → uint8 → SIFT (128-D descriptors) → brute-force L2, k=2 NN, Lowe ratio 0.75 → RANSAC homography (min 8 pts, 3.0 px reprojection threshold) → degeneracy check → best tile-pair = highest inlier count.
- WHY IT CONNECTS: Directly addresses multi-sensor, illumination-invariance, and outlier-rejection requirements from our problem statement.
- WHAT WE COULD USE: Full pipeline structure — tiling → normalization → SIFT → ratio test → RANSAC → GCP extraction → declustering → Z-score filter → polynomial warp.
- Evidence: [DEMONSTRATED] Mean RMSE ~73 m across 8 detailed strips (of ~200 processed); 5/8 below single-pixel resolution (~80 m), all 8 <85 m.
- Implementation note: Per-tile independent processing handles scene heterogeneity (craters vs. maria) — useful for OHRC where texture varies strongly within a strip.

### Radiometric Normalization for Cross-Sensor Matching
- WHAT: Percentile clip + min-max scale + mean/std statistic-transfer of source patch to reference patch before descriptor computation (skipped if reference std ≈ 0).
- WHY IT CONNECTS: Multi-modal correspondence (OHRC/TMC/IIRS vs. reference) needs radiometric harmonization since raw appearance differs across sensors — a stated project challenge.
- WHAT WE COULD USE: As a pre-processing stage before any detector (SIFT/ORB/learned) for cross-band matching (e.g., OHRC↔TMC).
- Evidence: [REPORTED] Used prior to SIFT; contributes to matching under differing sensor radiometry (not isolated/ablated).
- Implementation note: Needs adaptation for sensor pairs with far larger scale gaps than IIRS/WAC's ~1.25x (e.g., OHRC 0.25–0.5 m vs. IIRS 80 m).

### GCP Extraction, Declustering, and Outlier Filtering
- WHAT: Convert inlier matches to GCPs (pixel + lon/lat), enforce min 15–20 px spacing (grid-based, keep point nearest cell center), remove outliers via residual Z-score (Eq.3; requires >20 GCPs).
- WHY IT CONNECTS: Directly satisfies our "uniform match distribution" and "outlier rejection" requirements.
- WHAT WE COULD USE: Exact declustering + Z-score filter as a standard post-processing module, detector-agnostic.
- Evidence: [DEMONSTRATED] Used operationally across ~200 strips; contributes to final sub-pixel RMSE.
- Implementation note: Min-GCP threshold and spacing are tunable — may need recalibration for OHRC's higher resolution/smaller footprints.

## 3. P1 — HIGHLY RELEVANT / INTEGRABLE

### RANSAC Homography as Verification (vs. Final Polynomial Warp)
- WHAT: Homography used only for match verification (inlier selection); final warp uses a separate polynomial fit to filtered GCPs.
- WHY RELEVANT: Suggests decoupling local geometric verification (per-tile homography) from global warp (polynomial) — useful separation for us.
- HOW IT COULD INTEGRATE: Use homography-RANSAC for tile-level filtering; use higher-order polynomial/piecewise model for final OHRC/TMC push-broom strip geometry (along-track distortions homography can't capture).
- WHAT WOULD NEED TO CHANGE: Validate tile size vs. local terrain relief, since homography assumes a single planar transform per tile.
- ADVANTAGE: Simple, cheap, works with sparse GCPs (min 8 points).
- CHALLENGE: Degenerates under significant local relief (craters, highlands) — not tested in this paper.
- OPPORTUNITY: Benchmark homography vs. affine vs. thin-plate-spline per tile against our sub-pixel accuracy target.

## 4. P2 — RESEARCH THEORY / TECHNICAL INSIGHT

### Solar Incidence Angle Mismatch as Dominant Error Source
- CONCEPT: Incidence-angle mismatch between source/reference degrades SIFT correspondence even though SIFT is nominally illumination-invariant.
- ESTABLISHED: [DEMONSTRATED] WAC incidence angles 48°–72° (equator) vs. IIRS ~20°–43° (5°S–28°N); identified as likely main cause of the 65–85 m RMSE range — error ~60–90 m near equator, escalating to 240–300 m beyond ±40° latitude.
- WHY IT MATTERS: Falsifies the common assumption that SIFT's illumination invariance alone suffices for large solar-geometry gaps — directly tests our problem statement's Sun-angle robustness requirement.
- PRACTICAL IMPLICATION: [INFERENCE] Expect proportionally larger error when incidence-angle difference exceeds ~20–30°; may need explicit angle-aware preprocessing (shadow removal, photometric correction, angle-matched reference selection) beyond relying on SIFT invariance.

### Latitude / Lunar Curvature as Error Amplifier
- CONCEPT: Beyond ±55° latitude, lunar curvature amplifies error independent of illumination.
- ESTABLISHED: [DEMONSTRATED] RMSE rises from ~65–80 m (low/mid latitude) to 1000–2000+ m near ±60–70° (Fig. 5, 3 strips), near-exponential trend.
- WHY IT MATTERS: Exposes a geometric (not just radiometric) limit of simple polynomial/homography warping — relevant since our imagery may include high-latitude/polar regions.
- PRACTICAL IMPLICATION: [INFERENCE] A single global transform (or per-tile homography) is inadequate near poles; likely needs rigorous sensor/orbital geometry modeling (e.g., bundle adjustment, map-projection-aware warp) — an explicit author-acknowledged limitation.

## 5. P3 — EXISTING / PAST SOLUTIONS

### Metadata-Based Seleno-Referencing (ref. [2])
- APPROACH: Use IIRS XML metadata (lat/lon at 50-px intervals) to geometrically correct strips directly — no image matching.
- RESULT: [REPORTED] Produces offset error since metadata lacks accurate pixel-to-pixel coordinates (sparse sampling).
- STRENGTH: Fast, no reference image needed.
- LIMITATION: Accumulates pointing/orbit/sensor-model errors; sparse sampling causes systematic offset.
- RELEVANCE TO US: Effectively a P4 case — should not be relied on standalone; image-to-image correspondence outperforms it.

### Manual GCP Extraction
- APPROACH: Human-picked GCPs between IIRS and LRO-WAC.
- RESULT: [REPORTED] Tedious, slow, error-prone (offsets of several meters).
- LIMITATION: Not scalable to large strip counts.
- RELEVANCE TO US: Confirms the need for an automated software solution, matching our deliverable requirement.

## 6. P4 — WHAT NOT TO TRY / FAILURE MODES

### Correlation-Based (Intensity/Template) Matching
- WHY USED: Simple, cheap, classical.
- FAILURE: [REPORTED, ref. 8] Fails on lunar terrain due to uniform albedo and subtle topography — low contrast + illumination-dependent appearance reduces correlation reliability.
- CONDITIONS: Especially over maria (low relief/albedo contrast) and under differing illumination geometry.
- IMPLICATION: Confirms our problem statement's premise — prioritize feature-based/invariant descriptors over raw intensity correlation.
- ALTERNATIVE: SIFT or other scale/illumination-invariant local descriptors.

### Relying on SIFT Invariance Alone Under Large Angle Differences
- WHY USED: SIFT is assumed illumination-invariant.
- FAILURE: [DEMONSTRATED] Error scales with incidence-angle mismatch (up to 3x pixel size beyond ±40° latitude, where mismatch is largest).
- WHY IT FAILS: Shadow/shading gradient shifts keypoint locations/descriptors under differing illumination, causing subtle misidentification even post ratio-test.
- CONDITIONS: Incidence-angle differences >~20–30°, and/or latitude >±40–55°.
- IMPLICATION: Don't treat SIFT illumination invariance as sufficient alone; supplement with shadow-aware preprocessing or a more robust descriptor.
- ALTERNATIVE: Photometric correction pre-step (paper's ref. [3]) or illumination-invariant descriptors (ref. [8] — priority next read).

## 9. SOLUTION / DECISION MATRIX

| Finding / Method | What it solves | Why useful | Why NOT use as-is | Challenge | Opportunity | Relevance |
|---|---|---|---|---|---|---|
| Tiled SIFT + Lowe ratio + RANSAC homography | Correspondence + outlier rejection | Proven on same spacecraft, sub-pixel RMSE | Homography assumes local planarity; may not suit large OHRC scale gaps | Robustness to large incidence-angle gaps | Extend to OHRC/TMC pairs, swap descriptor | P0 |
| Patch statistic normalization | Cross-sensor radiometric mismatch | Simple, effective baseline | Not a true photometric model; limited under large angle gaps | Scaling to very different resolution ratios | Combine with photometric correction | P0/P1 |
| GCP declustering + Z-score filter | Uniform distribution + outlier removal | Directly meets our requirements | None significant | Tuning spacing/threshold per sensor | Reuse directly | P0 |
| Metadata-only georeferencing | Fast registration, no image matching | N/A (avoid) | Sparse metadata → systematic offset | — | Use only as coarse seed for refinement | P4 |
| Correlation/template matching | Classical registration | N/A (fails on lunar terrain) | Fails under low texture/illumination differences | — | — | P4 |
| Global polynomial warp | Final geometric transform | Simple, works <±55° latitude | Breaks down near poles/high relief | Need higher-order/local model for edge cases | Explore piecewise/rational models | P1/P5 |

## 10. KEY TAKEAWAYS FOR OUR PROJECT
1. [USE NOW] Tiled SIFT → radiometric normalization → ratio test → RANSAC homography → GCP declustering → Z-score filter → polynomial warp is a strong, transferable baseline architecture for OHRC/TMC/IIRS correspondence.
2. [USE NOW] Benchmark: mean RMSE ~73 m (sub-single-pixel) across 200 IIRS strips vs. LRO-WAC — useful comparative accuracy target, though our sensors (esp. OHRC) differ greatly in resolution.
3. [AVOID] Correlation/template matching (fails on low-texture, uniform-albedo terrain) and metadata-only georeferencing (systematic offsets) as standalone methods.
4. [CRITICAL CONSTRAINT] Sun-angle mismatch is the dominant demonstrated error driver — SIFT's illumination invariance alone is insufficient under large angle gaps (error scales ~1x to ~3x pixel size). Must be explicitly addressed, not assumed solved.
5. [CRITICAL CONSTRAINT] Latitude/lunar curvature causes near-exponential error growth beyond ±55° — pure 2D transforms (homography/polynomial) are geometrically insufficient at high latitude.
6. [RESEARCH GAP] No sensor pair here has our extreme scale disparity (e.g., OHRC ~0.25 m vs. IIRS 80 m) — tile-based statistic-matching is untested at such gaps and needs validation.
7. [INVESTIGATE NEXT] Ref. [8] ("Illumination invariant feature point matching for high-resolution planetary remote sensing images") directly targets the illumination-robustness gap this paper identifies but doesn't solve — high priority for next review.
8. [METRIC PRECEDENT] RMSE_x/y/total (meters) and inlier-count-based tile selection are directly usable evaluation metrics, aligning with SIH's RMSE/inlier-ratio requirements.
