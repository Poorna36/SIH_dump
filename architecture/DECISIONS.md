# SIH26166 — DECISIONS (Design Decision Log, v1.0)

Documents all key architectural and design decisions made for the SIH26166 project, with particular focus on the Matcher Selection Model (MSM / L1.5) introduced in v2.0. Every entry follows the Architecture Decision Record (ADR) format.

Related docs: ARCHITECTURE.md, PIPELINE.md, FEATURES.md, CONFIGURATION.md, INTERFACES.md, VALIDATION.md, IMPLEMENTATION_PLAN.md.

---

## Index of Decisions

- **DEC-001**: Benchmark-First, Pluggable Matcher Engine Architecture
- **DEC-002**: Two-Stage Spatial Selection (ANMS + Coverage Grid)
- **DEC-003**: Geodetic Control Network & Padded Reference Patching Strategy
- **DEC-004**: Matcher Selection Model (MSM) Position in Pipeline (L1.5)
- **DEC-005**: LightGBM as the MSM Classifier Framework
- **DEC-006**: Dual Confidence Threshold Routing & Escalation Fallback Policy
- **DEC-007**: Strict Geo-Cell Disjointness for MSM Training (F15 Leakage Rule)
- **DEC-008**: Operational Gate for MSM Production Activation

---

## DEC-001: Benchmark-First, Pluggable Matcher Engine Architecture

* **Status**: Accepted
* **Date**: 2026-08-31
* **Context**: SIH26166 requires matching Chandrayaan-2 imagery (OHRC, TMC-2, IIRS) against LRO/SELENE reference basemaps across diverse lunar terrains, latitudes, and sun angles. Literature shows no single matcher performs best across all lunar conditions (SIFT fails at poles, RIFT is slow, SuperPoint+LightGlue is learned, Crater matching requires dense craters).
* **Decision**: Implement a pluggable matcher interface (`Matcher`) where all matchers run through the exact same spatial selection (L3), geometric verification (L4), sub-pixel refinement (L5), and evaluation harness (L7). The winning matcher per pair/stratum is determined empirically by the benchmark, not hardcoded upfront.
* **Consequences**:
  * Clean abstraction allows introducing/comparing matchers without changing core pipeline code.
  * Fair benchmarking because L3–L7 post-processing is identical across matchers.
  * Higher compute cost if all matchers run on every pair in production (addressed by DEC-004).

---

## DEC-002: Two-Stage Spatial Selection (ANMS + Coverage Grid)

* **Status**: Accepted
* **Date**: 2026-08-31
* **Context**: Problem statement explicitly demands uniform spatial distribution of match points across the overlap area rather than dense clustering on high-contrast feature hubs (e.g., large crater rims).
* **Decision**: Adopt a dual-stage selection approach:
  1. Keypoint-level Adaptive Non-Maximal Suppression (ANMS using Suppression via Square Covering / SSC) applied immediately after detection for sparse matchers (M0/M1).
  2. Match-level coverage-aware grid selection (N×N grid with per-cell capping and bisection thresholding) applied after matching (L3) for all matchers.
* **Consequences**:
  * Ensures spatial distribution is enforced both at detection time and match-selection time.
  * Introduces `spatial_coverage` and `grid_density_std` as mandatory first-class metrics in the evaluation harness.

---

## DEC-003: Geodetic Control Network & Padded Reference Patching Strategy

* **Status**: Accepted
* **Date**: 2026-08-31
* **Context**: Chandrayaan-2 initial ephemeris/pointing metadata contains positional uncertainty (up to several hundred meters to kilometers). Searching global reference basemaps blindly is computationally intractable.
* **Decision**: Use ISIS3 `spiceinit` / ASP SPICE kernels for initial coarse localization. Expand the metadata bounding box by 2–5× pointing uncertainty ($k \cdot \sigma_{\text{pointing}}$) to crop a padded reference patch from LROC WAC/NAC tiles via GDAL/WMTS. Run feature matching strictly within the padded crop.
* **Consequences**:
  * Guarantees true corresponding ground region is contained within the search crop.
  * Reduces reference patch search area to manageable dimensions (~few kilometers), preserving speed.

---

## DEC-004: Matcher Selection Model (MSM) Position in Pipeline (L1.5)

* **Status**: Accepted
* **Date**: 2026-08-31
* **Context**: Running all 4 matchers (M0 SIFT, M1 RIFT2, M2 SuperPoint+LightGlue, M3 Crater) on every pair in production incurs a 4× runtime penalty compared to running only the best matcher for that pair.
* **Decision**: Introduce a lightweight Matcher Selection Model (MSM) as layer **L1.5** between Preprocessing (L1) and the Correspondence Engine (L2). The selector uses 13 early-available, cheap features (sensor pair, GSD ratio, latitude, solar angle difference, terrain class, crater density, texture/contrast statistics, etc.) to predict which single matcher will perform best.
* **Consequences**:
  * Reduces average processing time by ~50–75% in production mode.
  * Preserves full multi-matcher execution as an available benchmark mode (`msm.enabled: false`).
  * Requires a training pipeline (`train_msm.py`) and explicit validation protocol before activation.

---

## DEC-005: LightGBM as the MSM Classifier Framework

* **Status**: Accepted
* **Date**: 2026-08-31
* **Context**: The selector needs to perform multi-class classification (4 classes corresponding to matchers) on a mix of continuous and categorical tabular features with sub-millisecond latency on CPU.
* **Decision**: Select **LightGBM** (via scikit-learn `LGBMClassifier` wrapper) as the model framework for MSM.
* **Rationale**:
  * Handles mixed categorical (`sensor_pair_enc`, `terrain_class_enc`) and continuous features natively without required manual one-hot encoding.
  * Invariant to monotonic feature scaling, reducing pre-processing overhead.
  * Inference latency < 1 ms on standard CPU.
  * Provides interpretable feature importance metrics (split & gain).
* **Consequences**:
  * Requires `lightgbm` dependency in the python/conda environment.
  * Simple model serialization via standard pickle/joblib pipelines.

---

## DEC-006: Dual Confidence Threshold Routing & Escalation Fallback Policy

* **Status**: Accepted
* **Date**: 2026-08-31
* **Context**: ML predictions can be uncertain on edge-case lunar scenes or unseen environmental combinations. Misrouting to a failing matcher without fallback would degrade registration reliability.
* **Decision**: Implement a two-tiered confidence thresholding mechanism:
  1. **High Confidence ($\ge \tau_{\text{high}} = 0.65$)**: Execute only the predicted `selected_matcher`.
  2. **Moderate Confidence ($\tau_{\text{low}} \le \text{confidence} < \tau_{\text{high}}$, $\tau_{\text{low}} = 0.40$)**: Execute `fallback_matcher` (second-highest probability).
  3. **Low Confidence ($< \tau_{\text{low}}$)**: Revert to full multi-matcher execution mode (`very_low_confidence_full_mode`).
  4. **Post-routing Gate Failure**: If the routed matcher fails the S4 match count gate ($< 150$ candidates), automatically escalate to M0 SIFT as safe fallback and log to `msm_fallback.jsonl`.
  5. **Hard Rule Gating**: M3 Crater is forced to probability 0 if crater density $< \tau_c$; M2 LightGlue is forced to 0 if GPU is unavailable.
* **Consequences**:
  * Prevents pipeline failures or registration loss on difficult/unusual image pairs.
  * Lowers risk of automation failures to near zero by incorporating defensive fallbacks at every stage.

---

## DEC-007: Strict Geo-Cell Disjointness for MSM Training (F15 Leakage Rule)

* **Status**: Accepted
* **Date**: 2026-08-31
* **Context**: Standard random pair splitting leads to spatial leakage if overlapping or adjacent image pairs end up in both training and test sets, artificially inflating benchmark metrics.
* **Decision**: Formalize Fix F15: MSM training labels are derived **strictly from pairs in the training split** of the geo-cell partition ($10^\circ \times 10^\circ$ geographic tiles). No pair from a test geo-cell can be used for MSM model fitting. Extended leakage verification is enforced via `leakage_audit --check-msm`.
* **Consequences**:
  * Ensures selector accuracy metrics reflect true generalization to unseen lunar regions.
  * Prevents overoptimistic performance estimation during evaluation.

---

## DEC-008: Operational Gate for MSM Production Activation

* **Status**: Accepted
* **Date**: 2026-08-31
* **Context**: MSM must not be activated in production prematurely without verifying its efficacy on held-out test data.
* **Decision**: `msm.enabled` in `configs/msm.yaml` defaults to `false` and can only be set to `true` after passing all 8 Acceptance Criteria (AC1–AC8) in `VALIDATION.md §2.3`:
  * Selector accuracy $\ge 70\%$
  * Top-2 accuracy $\ge 85\%$
  * Mean RMSE degradation $\le +0.10$ px
  * Max single-pair RMSE degradation $\le +0.50$ px
  * Runtime reduction $\ge 50\%$
  * Fallback rate $\le 20\%$
  * Feature importance non-trivial
  * Leakage audit passes (`exit 0`)
* **Consequences**:
  * Ensures production deployment is strictly evidence-driven.
  * System safely defaults to full multi-matcher benchmarking when training data is insufficient.
