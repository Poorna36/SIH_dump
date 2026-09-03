# SIH26166: Multi-Modal Lunar Image Registration — Research Dossier & System Reference

**Task:** Autonomous, sub-pixel image-to-image correspondence between Chandrayaan-2 payloads (OHRC, TMC-2, IIRS) and lunar reference basemaps (LROC NAC, WAC 643nm, SELENE TC).

---

## 1. Operating Regime & Physical Constraints

Standard terrestrial and planetary feature matchers fail under lunar surface imaging conditions due to four distinct physical mechanisms:

- **Illumination-Driven Gradient Inversion:** Solar incidence ($\theta_{\text{inc}} \in [15^\circ, 85^\circ]$) and azimuth offsets ($\Delta\phi_{\text{sun}}$ up to $180^\circ$) shift shadow boundaries across crater rims. Intensity gradient vectors $\nabla I$ rotate or reverse direction between passes. Classical DoG/Hessian descriptors (SIFT, SURF, ORB) decorrelate once $\Delta\phi_{\text{sun}} > 30^\circ$, dropping to $0\%$ inliers in polar regions.
- **Cross-Payload Scale Disparity:** Ground Sampling Distance (GSD) varies by more than two orders of magnitude across sensor pairs:
  - OHRC ($0.25\text{ m}$) vs. LROC NAC ($0.5-1.5\text{ m}$): $2\times - 6\times$ scale ratio.
  - TMC-2 ($5.0\text{ m}$) vs. LROC WAC ($100\text{ m}$): $20\times$ scale ratio.
  - OHRC ($0.25\text{ m}$) vs. LROC WAC ($100\text{ m}$): $400\times$ scale ratio (precludes direct single-stage matching; requires intermediate TMC-2 or tiled pyramid bridging).
- **Topographic Shadow Occlusion:** Near the lunar poles ($|\text{lat}| > 70^\circ$), low sun elevation ($< 2^\circ$) casts deep shadows with near-zero radiometric contrast ($\sigma_I^2 < 10$). Feature detectors placed inside shadowed terrain return noise or unstable edge responses.
- **Uniform Point Distribution:** Downstream map-projection and bundle adjustment algorithms require tie points distributed across the full image footprint, not clustered exclusively on high-contrast crater rims.

---

## 2. Empirical Benchmark & Method Comparison

Based on empirical evaluations across Chandrayaan-2 and LROC test datasets:

| Method | Descriptor Type | Scale Ratio Limit | Max Azimuth Offset ($\Delta\phi$) | Reported RMSE | Primary Failure Mode | Technical Reference |
|---|---|---|---|---|---|---|
| **SIFT** | DoG + Gradient Hist (128-D) | $5.0\times$ | $\approx 25^\circ$ | $1.25\text{ px}$ | Complete failure under polar shadow shifts | [sift_iirs.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/sift_iirs.md) |
| **I-KAZE** | Non-linear Diffusion + PC | $2.0\times$ | $\approx 45^\circ$ | $0.64\text{ px}$ | Fails on cross-modal radar/optical pairs | [kaze_diffusion.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/kaze_diffusion.md) |
| **RIFT / RIFT2** | Phase Congruency + MIM | None ($1.0\times$ via GSD resample) | $\approx 90^\circ$ | $1.85\text{ px}$ | High runtime; requires external scale normalization | [rift_phase_congruency.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/rift_phase_congruency.md) |
| **CNSFM** | Crater Centroid Triples | $10.0\times$ | $\approx 150^\circ$ | $1.20\text{ px}$ | Zero matches on smooth maria ($\rho_c < 5/\text{tile}$) | [cnsfm_crater_matching.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/cnsfm_crater_matching.md) |
| **SuperGlue** | SuperPoint + Attentive GNN | $3.0\times$ | $\approx 180^\circ$ | $0.35\text{ px}$ | Requires GPU; drops matches in textureless plains | [superglue_gnn.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/superglue_gnn.md) |
| **LoFTR** | Transformer Coarse-to-Fine | $2.0\times$ | $\approx 120^\circ$ | $0.28\text{ px}$ | Extrapolates out-of-bounds coords in shadows | [loftr_transformer.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/loftr_transformer.md) |
| **Phase Corr.** | Fourier Cross-Power Spectrum | $1.1\times$ | $\approx 3^\circ$ | $0.01\text{ px}$ | Narrow convergence basin; translation only | [phase_correlation.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/phase_correlation.md) |

---

## 3. Core Technical Decisions

1. **Illumination Robustness Strategy:**
   - On cratered terrain: Prioritize learned attention (SuperGlue/LightGlue) or topological crater matching (CNSFM).
   - On smooth terrain / maria: Use Wallis adaptive filtering ($W=64\text{ px}$) to enhance subtle albedo differences before computing phase congruency (RIFT2) or SIFT.
2. **Sub-Pixel Precision via Dual-Stage Execution:**
   - Wide-baseline search (L2) establishes integer correspondences at $1-2\text{ px}$ precision.
   - Refinement pass (L5) fits a 2D paraboloid over a localized $3\times 3$ phase correlation cross-power peak, tightening residual error to $<0.1\text{ px}$.
3. **Outlier Filtering & Degeneracy Safeguards:**
   - DEGENSAC replaces standard RANSAC to prevent degenerate plane fits over flat lunar maria.
   - Candidate matches from learned models undergo strict coordinate domain verification: $(x, y) \in [0, W] \times [0, H]$.
4. **Dynamic Matcher Arbitration (MSM / L1.5):**
   - A LightGBM classifier routes incoming image pairs to the optimal matcher based on 13 lightweight metadata and image statistics features (GSD ratio, $|\text{lat}|$, $\Delta\phi_{\text{sun}}$, crater density $\rho_c$, shadow fraction).
   - High confidence ($P \ge 0.65$): Runs single selected matcher.
   - Fallback threshold ($0.40 \le P < 0.65$): Runs secondary fallback matcher.
   - Low confidence ($P < 0.40$): Falls back to full benchmark multi-matcher execution.

---

## 4. Repository Index & Module Structure

### Research & Method Analysis (`research/`)

- [phase_correlation.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/phase_correlation.md): Normalized cross-power spectrum, Gaussian apodization windowing, and 2D paraboloid peak fit equations.
- [cnsfm_crater_matching.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/cnsfm_crater_matching.md): Invariant structural triangles of crater centroids; south pole illumination benchmark.
- [superglue_gnn.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/superglue_gnn.md): GNN self/cross-attention message passing, Sinkhorn optimal transport, and dustbin formulation.
- [loftr_transformer.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/loftr_transformer.md): Detector-free coarse matching via dual softmax, fine heatmap regression, and out-of-domain sanity checks.
- [rift_phase_congruency.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/rift_phase_congruency.md): Log-Gabor filter phase congruency, Maximum Index Map (MIM) construction, and multi-orientation search.
- [sift_iirs.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/sift_iirs.md): Scale-space DoG matching for Chandrayaan-2 IIRS vs. LRO WAC; latitude error degradation curves.
- [desca_contour.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/desca_contour.md): Differential Evolution sample consensus for affine model estimation; two-tier match set initialization.
- [kaze_diffusion.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/kaze_diffusion.md): AOS non-linear diffusion filtering, phase congruency keypoint weighting, and mutual information fine alignment.
- [subpixel_refinement.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/subpixel_refinement.md): 2D peak interpolation models and least-squares covariance propagation.
- [radiometric_norm.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/radiometric_norm.md): Wallis adaptive contrast filtering and Lommel-Seeliger photometric correction models.
- [comparative_taxonomy.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/research/comparative_taxonomy.md): Unified trade-off matrix covering latency, memory, hardware constraints, and invariant limits.

### System Architecture & Pipeline (`architecture/`)

- [MASTER_SPECIFICATION.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/architecture/MASTER_SPECIFICATION.md): End-to-end system design, L0–L5 layers, the 15 architectural principles ($F1$–$F15$), and MSM router specification.
- [PIPELINE.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/architecture/PIPELINE.md): Execution runbook covering stages S0 through S9, CLI scripts, and gating criteria.
- [INTERFACES.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/architecture/INTERFACES.md): Python interface contracts and dataclass schemas (`PairRecord`, `MatcherResult`, `SelectorResult`).
- [FEATURES.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/architecture/FEATURES.md): Definitions and extraction logic for the 13 MSM routing features.
- [CONFIGURATION.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/architecture/CONFIGURATION.md): System hyperparameters and operational profiles.
- [DECISIONS.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/architecture/DECISIONS.md): Architectural Decision Records (ADRs DEC-001 through DEC-008).
- [VALIDATION.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/architecture/VALIDATION.md): Benchmark protocol, geographic cell-disjoint splitting rules, and unit test suites.

### Reference Data & Sensors (`references/`)

- [reference_mapping.md](file:///d:/neo/hachathon/SIH%202026/SIH_DUMP/references/reference_mapping.md): Instrument parameters for OHRC, TMC-2, IIRS, LRO NAC/WAC, and bounding box padding calculations.
