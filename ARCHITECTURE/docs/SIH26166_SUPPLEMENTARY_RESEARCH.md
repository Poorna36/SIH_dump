# SIH 26166 — Supplementary Research (Gaps 3 & 4 + Cross-Sensor Family)

---

## 1. Uniform Spatial Distribution of Matches

Your PS explicitly requires matches spread evenly across the image, not clustered in high-texture regions (e.g. crater clusters). Nobody in your current reading list addresses this — it's a keypoint-selection problem, solved independently of which detector/descriptor you pick.

Core technique: Adaptive Non-Maximal Suppression (ANMS)
- Standard reference: *"Efficient adaptive non-maximal suppression algorithms for homogeneous spatial keypoint distribution"* — Bailo et al., Pattern Recognition Letters, 2018.
- Idea: instead of keeping the top-N strongest keypoints (which clusters around a few high-contrast craters), each keypoint is suppressed if a *stronger* keypoint exists within some radius. The radius adapts per-point so selected points are spread out but still locally strongest.
- 3 variants proposed (SSC, KDT-inspired, RT): SSC (Suppression via Square Covering) is fastest and scales best — use this one.
- Ready-to-use code: `github.com/BAILOOL/ANMS-Codes` — C++, MATLAB, and Python implementations, drop-in after any OpenCV detector (SIFT/ORB/AKAZE keypoints → ANMS → filtered set).
- Directly used inside LNIFT itself (see §3) to get "evenly distributed features" — so if you adopt LNIFT you get this for free; if you use RIFT/SIFT/etc. you'll need to bolt ANMS on separately.

Complementary technique: Grid/tile-based enforcement
- Simple alternative if ANMS is too slow for your pipeline: divide the image into an N×N grid, cap keypoints per cell (e.g. top-k by response), run RANSAC per neighborhood or globally after this cap. Cruder than ANMS but trivial to implement and easy to justify in a report.
- Ames Stereo Pipeline (see §2) already uses a tile-based approach internally (`--ip-per-tile`) for exactly this reason — worth citing as a real flight-software precedent.

Where this fits your pipeline: after keypoint detection, before descriptor matching. Order: detect → ANMS filter → describe → match → RANSAC/geometric outlier rejection. Report both *before* and *after* spatial-distribution metrics (e.g. std-dev of point density across grid cells) as one of your evaluation figures — this is a cheap, concrete number the PS explicitly asks you to demonstrate.

---

## 2. Software Tooling for Lunar Data — ASP + ISIS3

These are not papers — they're the actual infrastructure to ingest, calibrate, and geometrically process Chandrayaan-2 and LRO data before your matching algorithm ever runs. Treat this as your data pipeline layer, separate from your algorithm layer.

### USGS ISIS3 (Integrated Software for Imagers and Spectrometers)
- The planetary-science standard for calibrating raw orbital imagery: radiometric correction, SPICE-based camera geometry (`spiceinit`), map projection/orthorectification.
- You need this before feature matching — raw PDS4 `.img`/`.xml` products from OHRC/TMC/IIRS are not directly comparable pixel grids; ISIS3 converts them into calibrated `.cub` files with known geometry.
- Install note: ships as part of the `usgs-astrogeology` conda channel. Recent dev builds (2026.06.07+) needed for compatibility with newer ASP versions — check version alignment before installing.

### NASA Ames Stereo Pipeline (ASP)
- Documentation includes a worked example specifically using Chandrayaan-2 OHRC and TMC-2 data (`stereopipeline.readthedocs.io/en/latest/examples/chandrayaan2.html`) — this is close to a reference implementation for your exact data sources.
- Workflow relevant to you:
  1. `isisimport`: converts raw `.xml`/`.img` PDS4 product → `.cub` file.
  2. `spiceinit` / `isd_generate`: attaches camera geometry (needs valid ISIS/ALE/USGSCSM versions).
  3. Interest-point matching + bundle adjustment across image pairs — ASP does its own SIFT-like interest point detection with `--ip-per-tile` (tile-based, i.e. already spatially distributed) as a preprocessing step for stereo, which is functionally close to what your PS wants, just for a different downstream purpose (DEM generation vs. general correspondence).
  4. `parallel_stereo` with `--alignment-method local_epipolar --stereo-algorithm asp_mgm --subpixel-mode 9`: note the explicit subpixel-mode flag — ASP's built-in subpixel refinement (mode 9) is directly relevant to your sub-pixel accuracy requirement and worth reading as a concrete method, not just a black box.
  5. The docs explicitly flag that TMC-2 ortho-mosaic illumination differs drastically from OHRC strip images — this is ISRO/USGS-acknowledged evidence that illumination variation between your two primary sensors is a real, documented problem, not a hypothetical.
- Practical framing for your report: you don't need to reimplement calibration/orthorectification/camera geometry from scratch — ASP + ISIS3 handles that. Your novel contribution is the correspondence/matching algorithm layered on top of ASP/ISIS3-calibrated imagery, plus your own evaluation (RMSE, inlier ratio, uniform distribution). This is a strong feasibility argument for your SIH pitch: "we build on flight-proven USGS/NASA infrastructure, our contribution is the matching layer."
- Requires environment care: ASP 3.7.0's bundled conda environment is the tested baseline; mixing a separately-installed ISIS with ASP is a documented source of errors.

---

## 3. Cross-Sensor / Multimodal Technique Family

These are the hand-crafted algorithms designed for the exact problem your PS calls "multi-modal correspondence" — matching images with different intensity/radiometric behavior (SAR-optical originally, but the same nonlinear radiation distortion (NRD) problem applies to OHRC vs TMC vs IIRS, which differ in resolution, spectral band, and sensor type).

| Method | Year | Core idea | Speed vs RIFT | Notes for your use case |
|---|---|---|---|---|
| OS-SIFT | 2018 | Multi-scale Sobel gradients on optical, ROEWA operator on SAR, to make gradients comparable across modalities | baseline | Detector-side fix only; good as a comparison baseline, not likely your primary method since OHRC/TMC/IIRS aren't SAR |
| CFOG (Channel Features of Oriented Gradients) | 2019 | Pixel-wise HOG-like channel descriptor + structural similarity metric instead of raw intensity/NCC | faster than RIFT | Template-matching style (dense, not sparse keypoints) — useful if you want a dense correspondence field rather than sparse tie-points |
| RIFT | 2020 | Phase congruency for detection (frequency domain) + Maximum Index Map descriptor from log-Gabor filters | slow (~47.8s/1024px img) | Your existing strongest multimodal reference; robust but computationally heavy — flag this tradeoff explicitly if you use it |
| RIFT2 | 2023 | Same detector, faster rotation-invariance technique (avoids convolution sequence ring) | ~3x faster than RIFT | Straightforward upgrade if you cite RIFT — mention this as "future work" or use directly |
| LNIFT | 2022 | Local normalization filter converts both modalities into a shared "intermediate modality" in the *spatial* domain (not frequency), then runs improved ORB + HOG descriptor, with ANMS built in | 0.49s vs RIFT's 47.8s on 1024px (≈100x), 99.9% vs 79.85% success rate, more correct matches (309 vs 119) | Strongest practical candidate for your project. Solves illumination/NRD robustness *and* uniform distribution (§1) *and* is near real-time — directly usable as your core method or strong baseline. Code: `github.com/LJY-RS/LNIFT_exe` |

Recommendation for your report: position LNIFT as your primary candidate method (spatial-domain, fast, has ANMS built in, rotation-invariant like ORB) with RIFT/RIFT2 and OS-SIFT/CFOG as your comparison baselines in the evaluation section. This gives you a clean "why we chose X over Y" narrative that SIH judges look for, backed by the actual published benchmark numbers above.

---

## 4. OHRC / TMC / IIRS — Sensor-Specific Notes (from search, beyond the 3 papers you'll read yourself)

- TMC-2: ~5 m resolution, broad coverage (>50% of lunar surface). Lower resolution than OHRC.
- OHRC: ~0.25–0.3 m resolution, but narrow-strip coverage (limited swath). This resolution gap (5m vs 0.3m, ~17-20x) is your core scale-variation challenge between these two sensors specifically — worth stating as a quantified number in your report rather than "large scale differences."
- IIRS: hyperspectral, visible + near-infrared. Co-registration challenges are fundamentally different from OHRC/TMC because you're matching *across spectral bands*, not just across spatial resolution/illumination — comparable prior work (SELENE Multiband Imager registration, 20–62m resolution) used manual tie-point selection + RANSAC-based affine + polynomial refinement rather than automated feature detectors, because spectral appearance differences are harder for SIFT-type detectors to bridge. If IIRS is in your prototype scope, this is a meaningfully different sub-problem — flag it as a stretch goal or separate module rather than assuming the same OHRC/TMC pipeline will work unmodified.
- Data access note: one existing paper (MoonMetaSync) sourced Chandrayaan-2 imagery from ISRO PRADAN servers, not the CHMAP link in your master doc — worth checking PRADAN as an alternate/additional data source when you get to actual dataset acquisition.

---

## 5. Sub-Pixel Refinement (brief — to address later as a whole)

Problem: Feature matching (SIFT/RIFT/LNIFT/etc.) gives you integer-pixel keypoint locations. Your PS requires sub-pixel accuracy, so you need a final refinement stage after coarse matching.

Possible solution: Parabolic/quadratic peak interpolation on the local correlation surface — after RANSAC gives you inlier matches, take a small correlation patch around each match, fit a parabola (2D or two 1D) to the peak, and the fitted vertex gives the true sub-pixel offset in closed form (cheap, no extra correlation cost). Reference: Pallotta et al., *"Subpixel SAR Image Registration Through Parabolic Interpolation of the 2-D Cross Correlation,"* IEEE TGRS 2020 — math is generic, not SAR-specific.
Ready-made implementations: `cv2.cornerSubPix()` (OpenCV, corner-style refinement) or `skimage.registration.phase_cross_correlation` (upsampled-DFT / Guizar-Sicairos method, sub-pixel translation). ASP's `--subpixel-mode 9` flag does something functionally similar internally.

---

## 6. Evaluation Methodology (brief — to address later as a whole)

Problem: You don't have a labeled test dataset with known "correct" answers for real lunar image pairs — there's no ground-truth transform to check RMSE against, unlike typical ML benchmarks.

Possible solution — two complementary approaches:
1. Held-out checkpoints (standard photogrammetric practice): From your matched points, set aside a subset (e.g. 20%) as checkpoints — never used to fit the RANSAC transform. Fit the transform on the remaining matches, then measure error only on the checkpoints. This avoids circularity (testing on the same points used to fit the model).
2. Synthetic ground truth (fallback / sanity check, easy to set up early): Take one real image, apply a *known* synthetic transform (rotation, scale, shift) to generate a fake "source" image. Now you know the exact ground-truth transform, so RMSE against it is exact. Good for week-1 pipeline testing before real cross-sensor pairs are working.

Metrics to report (all explicitly named in the PS):
- RMSE — root-mean-square distance between predicted and true/checkpoint positions, in pixels (report sub-pixel, e.g. 0.4 px).
- Inlier count — number of matches surviving RANSAC.
- Inlier ratio — inliers / total matches proposed (measures how "clean" your matcher is before geometric filtering).
- Spatial distribution metric (from §1) — e.g. std-dev of match density across a grid, to demonstrate the "uniform distribution" requirement, not just accuracy.
- Optional but easy to add: runtime per image pair (you already have LNIFT vs RIFT numbers to compare against).

---

## 7. IIRS — Registration-Specific Papers (kept in scope, lower priority)

IIRS is a genuinely different sub-problem from OHRC/TMC (hyperspectral, ~80 m/pixel, 250 spectral bands, 0.8–5.0 µm, no inherent map-projection info in the raw QUB files) — but here's what's directly usable when you get to it:

- "SIFT-Based Automated Registration of Chandrayaan-2 IIRS Hyperspectral Images" (2025) — already in your list, but note the key result: SIFT matching IIRS pixels against LRO-WAC (used as reference) achieves RMS error smaller than one IIRS pixel (~80 m). This is your direct accuracy benchmark to cite/beat if you build an IIRS module. https://sciety-labs.elifesciences.org/articles/by?article_doi=10.20944/preprints202511.1090.v1
- "Photometric correction of images obtained from Chandrayaan-2 IIRS data" (ScienceDirect) — necessary preprocessing step before any feature matching on IIRS: corrects for viewing-geometry differences (incidence/emission/phase angle) since IIRS images are acquired at varying geometry, similar in spirit to your illumination-invariance problem but solved radiometrically rather than algorithmically. https://www.sciencedirect.com/science/article/abs/pii/S0273117723009754
- Comparative Evaluation paper (already in your list, §Multimodal) confirms the standard fallback approach for IIRS co-registration when automated features fail: manual homologous point selection → initial affine transform → RANSAC refinement → polynomial correction. Useful to note as your "IIRS stretch-goal method" if full automated SIFT-based matching proves unreliable in your two weeks.
- Data source: IIRS radiance data is available via ISRO's Pradan archive (same source noted in §4), in PDS4-compliant QUB format.

Practical framing for your report: treat IIRS as "Phase 2 / future work" module with its own short methodology paragraph (photometric correction → SIFT-based registration against WAC, targeting sub-pixel/sub-80m RMSE) rather than folding it into your main OHRC/TMC pipeline — the preprocessing and reference image are both different, so pretending it's the same pipeline will undercut your report's credibility more than just scoping it honestly.

---

## 8. Suggested Method Stack (summary)

1. Calibration/geometry: ISIS3 (`spiceinit`, radiometric correction) → ASP (`isisimport`, orthorectification)
2. Core correspondence method: LNIFT (spatial-domain, fast, illumination-robust, built-in ANMS) — primary
3. Baselines for comparison: RIFT/RIFT2, OS-SIFT, CFOG, plain SIFT/ASIFT (as your existing papers already use)
4. Uniform distribution enforcement: ANMS (via LNIFT or bolted onto other detectors) or grid-based fallback
5. Outlier rejection: RANSAC (affine/homography), consider tile-based RANSAC per ASP precedent
6. Sub-pixel refinement: reference ASP's `--subpixel-mode 9` approach as a concrete method to adapt
7. Evaluation: RMSE (on held-out checkpoints, not fitting points), inlier count/ratio, spatial distribution metric (grid density std-dev), timing (given LNIFT vs RIFT speed gap is a real differentiator)
8. IIRS (lower priority): photometric correction → SIFT-based registration against LRO-WAC reference → target sub-80m (sub-pixel) RMSE, kept as a separate module/phase
