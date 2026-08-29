# SIH 26166 — Reference Image Mapping & Automated Patch Selection

---

## 1. How the mapping actually works, conceptually

Registration is not "compare two whole images." It's: take your moving/source image (Chandrayaan-2 OHRC/TMC/IIRS) and estimate a geometric transform (translation + rotation + scale, or affine/homography) that maps its pixels onto the fixed/reference image's coordinate system — using the correspondence points your feature-matching pipeline found. The reference image defines the trusted coordinate frame that everything else gets pulled into.

---

## 2. What LRO's map actually serves — it's the trusted frame, not just "another image"

- LOLA (Lunar Orbiter Laser Altimeter) laser-ranging measurements form the Moon's actual geodetic control network — the surveyed coordinate reference everything else is checked against. Accuracy is roughly tens of meters globally after crossover correction, better locally.
- LROC WAC global mosaic (~75–100 m/pixel, photometrically/illumination-normalized, built from 15,000+ images acquired over ~3 time windows to average out sun-angle drift) is tied to the LOLA geodetic frame. This is *why* it's the standard reference basemap the lunar-mapping community registers against — not because it's the sharpest imagery, but because its coordinates are independently trustworthy.
- LRO NAC (0.5–2 m) gives higher-resolution reference strips when you need finer detail matched against OHRC specifically (rather than the coarser WAC mosaic), but coverage is limited to narrow strips, not global.

## 3. Where SELENE fits

Kaguya's Terrain Camera (TC, ~10 m) and Multiband Imager (MI, 20–62 m) are supplementary reference sources:
- Useful where LRO coverage or acquisition geometry doesn't line up well with your target region.
- Useful as a second, independent reference to cross-check that your registration isn't overfitting to quirks of one specific mosaic.
- Precedent: the Chandrayaan-2 TMC cross-sensor radiometric-normalization paper already uses SELENE as an auxiliary reference alongside LROC WAC — same pattern you'd follow.

---

## 4. Automating source-to-reference patch selection (no manual work)

Key principle: the metadata footprint is a *search window*, not the final answer. Feature matching is what turns "roughly this region" into an exact sub-pixel alignment — the two steps are complementary, not redundant.

Steps:

1. Get the approximate footprint. Every Chandrayaan-2 product carries approximate corner lat/lon from spacecraft ephemeris. ISIS3's `spiceinit` step formalizes/refines this into a proper camera-geometry solution (see tooling report, §2).
2. Query the reference mosaic by bounding box — no manual browsing. NASA's Moon Trek WMTS API (`trek.nasa.gov`) serves LROC WAC/NAC tiles via bounding-box queries — feed it the footprint coordinates and it returns the matching tile(s) directly. Alternative: download the mosaic once locally and crop with GDAL/rasterio using the same bbox.
3. Pad the bounding box beyond the raw footprint. Rule of thumb: 2–5x the known pointing/ephemeris uncertainty (typically a few hundred meters to a couple km depending on processing level; OHRC's narrow-strip attitude solution tends to be less refined than TMC-2's full-frame photogrammetric processing). This guarantees the true corresponding region is inside your cropped reference patch even if the raw metadata is somewhat off.
4. Run feature matching only within the padded crop, not the full global mosaic. This is also what keeps the pipeline computationally feasible instead of searching the entire Moon per image.
5. Cross-check when possible. If both OHRC and TMC-2 cover the same target, TMC-2's generally better-refined ephemeris/geolocation can help sanity-check or bootstrap the coarse footprint used for the narrower, less-refined OHRC strip.

The coarse-to-fine logic in one line: metadata narrows planet-wide uncertainty down to a padded patch → feature matching narrows the patch down to sub-pixel alignment → LOLA/LROC WAC is why the final reported coordinates can be trusted as accurate, not just internally self-consistent.
