# ADDITIONAL RESOURCES — Papers, Tools, Repos, Products Connected to SIH 26166

Compiled via targeted web search to fill gaps identified across Papers 1–4 (esp. the unresolved "cross-sensor/multi-modal + Sun-angle" requirement). Organized by priority. Not full P0–P6 write-ups — flag items worth requesting a full breakdown on.

---

##  TOP PRIORITY — Directly on Chandrayaan-2 OHRC (read these first)

### 1. Sub-metre Lunar DEM Generation and Validation from Chandrayaan-2 OHRC Multi-View Imagery Using an Open-Source Pipeline (Aadi, Singla, Dube, Alexandrov — arXiv:2604.01032, 2026)
- Why critical: Uses the *exact* sensor named in our problem statement (OHRC). Builds sub-metre DEMs from non-paired OHRC archives by identifying candidate stereo pairs via metadata geometry (baseline-to-height ratio, convergence angle) — i.e., they solve a correspondence/registration problem on the *same imagery* SIH 26166 targets, using ASP + ISIS.
- Relevance: P0. Directly shows what "good" looks like on our actual sensor; their feature-matching/bundle-adjustment stage (inside ASP) is a working baseline to benchmark against or extract ideas from.

### 2. Geodetically Anchored 0.30m DEM of the Chandrayaan-3 Vikram Landing Site from OHRC Stereo Imagery (arXiv:2602.14993, 2026)
- Why critical: A second independent, fully open (ISIS + ASP + ALE) OHRC reconstruction, validated against LROC NAC stereo DEM (~30 m horizontal accuracy, 8.1 cm median triangulation error) — gives a concrete accuracy benchmark on the same instrument and a template for validating our own correspondence output against an independent reference (LRO NAC), exactly as the SIH brief allows.

### 3. DEM Refinement and Validation on the Lunar Surface Using Shape-from-Shading with Chandrayaan-2 OHRC Imagery
- Why relevant: From an ISRO/SAC (Space Applications Centre) team — shows an internal-to-ISRO methodology using shape-from-shading (photoclinometry) refinement on OHRC, built on the same ASP/ISIS open stack. Useful for understanding what ISRO's own community already does with this exact data, and as a possible complementary technique (photometric refinement) alongside correspondence-based registration.

### 4. Evaluation of image simulation open source solutions for simulation of synthetic lunar images (arXiv:2604.22296, 2026)
- Why relevant: Directly evaluates open-source tools for generating synthetic lunar images with controllable Sun-angle/illumination using real OHRC/LRO WAC/NAC-derived DEMs. This is a candidate solution to our biggest data gap: synthetic training/test data generation with controlled Sun-angle variation for a learned matcher, without needing scarce real multi-illumination lunar pairs.

---

##  HIGH PRIORITY — Fills the "multi-modal / illumination-invariant" gap directly

### 5. RIFT — Radiation-Variation Insensitive Feature Transform (Li, Hu & Ai, IEEE TIP 2020)
- What it is: A classical (non-learned) feature detector/descriptor purpose-built for multi-modal matching (optical–SAR, optical–infrared, optical–map, etc.) that is currently the most-cited, most widely reused baseline in this literature.
- Key idea directly useful to us: Uses phase congruency (not raw intensity/gradient) for keypoint detection, and a Maximum Index Map (MIM) built from log-Gabor filter responses for description — both are illumination/radiation-invariant by construction, unlike SIFT-style gradient descriptors.
- Why it matters for SIH 26166: This is a plausible, lightweight, no-training-data-required answer to the Sun-angle/illumination invariance requirement — complements the learned approaches (SuperGlue/LoFTR) already analyzed, and pairs naturally with Paper 4's phase-correlation framework (phase congruency and phase correlation are closely related frequency-domain tools).
- GitHub: `LJY-RS/RIFT-multimodal-image-matching` (official demo code).
- Suggest requesting a full P0–P6 breakdown of this one next — it's the strongest single candidate found for the illumination-invariance gap.

### 6. MatchAnything: Universal Cross-Modality Image Matching with Large-Scale Pre-Training (He et al., 2025, arXiv:2501.07556)
- What it is: Extends LoFTR/RoMa-style architectures with large-scale multi-domain pre-training specifically to achieve domain-agnostic matching "across diverse image types" — explicitly targets the cross-modality generalization gap that Papers 2–4 all flagged as unsolved.
- Why it matters: If this generalizes even partially to optical-optical cross-sensor (not just optical-SAR/medical), it could be the most direct existing answer to the OHRC↔TMC↔IIRS multi-modal requirement — worth testing off-the-shelf on lunar data before assuming we need to train from scratch.
- Relevance: P0/P1 candidate — recommend a dedicated deep-dive.

### 7. RoMa: Robust Dense Feature Matching (Edstedt et al., CVPR 2024) + RoMa v2 (2025)
- What it is: Dense (not just sparse) feature matcher combining frozen DINOv2 features (coarse) with a specialized CNN (fine detail) and a transformer match decoder; explicitly demonstrated to handle "extreme illumination" and "extreme scale and viewpoint" changes on the WxBS benchmark (36% mAA improvement over prior SOTA).
- Why it matters: Directly extends the illumination-robustness evidence chain from SuperGlue (Paper 3, day/night) — RoMa is reported to work under *more* extreme conditions than SuperGlue was tested on. Also produces dense (not sparse) correspondences, which may better satisfy the SIH "uniform distribution of matched points" requirement than sparse detector-based methods.
- GitHub: `Parskatt/RoMa` (MIT-licensed, includes a lightweight "Tiny RoMa" variant for constrained compute).

### 8. CM-Bench: Cross-Modal Feature Matching Benchmark Bridging Visible and Infrared Images (arXiv:2603.12690, 2026)
- What it is: A benchmark (not just a method) specifically for evaluating matchers (LoFTR, RoMa, MatchAnything, JamMa, DKM, etc.) on visible↔infrared cross-modal pairs.
- Why it matters: A ready-made template for how to *evaluate* cross-modal matchers rigorously — directly reusable for designing our own OHRC↔TMC / OHRC↔IIRS evaluation protocol, and its leaderboard tells us which matcher families are currently strongest on a genuinely different-modality task (closer to our problem than same-modality benchmarks like MegaDepth/ScanNet).

---

##  MEDIUM PRIORITY — Practical software/tooling to build on rather than reimplement

### 9. NASA Ames Stereo Pipeline (ASP) + USGS ISIS
- What they are: The community-standard, open-source, actively maintained stereo photogrammetry (ASP) and planetary image ingestion/camera-model (ISIS) software stack — already proven on OHRC, LRO NAC, HiRISE, and other planetary sensors (see Resources #1–3 above).
- Why it matters: Rather than building geometric/camera-model handling from scratch, our correspondence-finding software could sit on top of ISIS for camera-model/SPICE handling and feed matches into ASP's bundle-adjustment/triangulation stage — dramatically reducing the "practical software implementation" burden named in the SIH deliverable list.
- Relevance: Strong candidate for the "software/registered product" deliverable's underlying infrastructure, not just a reference.

### 10. hloc (Hierarchical-Localization toolbox, ETH Zürich CVG group)
- What it is: A modular, actively maintained Python toolbox that already wires together feature extraction (SuperPoint, DISK, etc.), matching (SuperGlue, LightGlue), and geometric verification/SfM (via COLMAP/pycolmap) into one pipeline — built by the same group as SuperGlue and LightGlue.
- Why it matters: Rather than reimplementing SuperGlue/LightGlue integration from scratch (as we discussed for Paper 3), hloc already provides this wiring, and is designed to be easy to extend with new detectors/matchers — realistic starting point for a working prototype pipeline.
- GitHub: `cvg/Hierarchical-Localization`.

### 11. LightGlue (Lindenberger, Sarlin & Pollefeys, ICCV 2023)
- What it is: SuperGlue's official, faster successor from the same lab — adaptive depth/width (early-stopping per image pair), ~16.9 ms for easy pairs, supports SuperPoint, DISK, ALIKED, and SIFT front ends, Apache-2.0 licensed, and now integrated into Hugging Face Transformers.
- Why it matters: If we adopt the SuperGlue-style architecture (Paper 3's P0 recommendation), LightGlue is very likely the better starting point in 2026 — same core idea, actively maintained, faster, more front-end options, and easier to deploy.
- GitHub: `cvg/LightGlue`.

### 12. DeepMoon (Silburt et al., companion to "Lunar Crater Identification via Deep Learning")
- What it is: An open-source (TensorFlow), trained-and-published CNN (U-Net-based) for detecting lunar crater positions/radii from DEM images, trained on LOLA/WAC DEM data, with 92% recovery of known craters and demonstrated transfer to Mercury.
- Why it matters: Directly enables the concrete idea proposed in Paper 4's analysis — replacing the Hough-line structure filter in a Fusion-style detector with a crater-rim structure filter for lunar imagery. This repo is a ready-made crater detector rather than something we'd need to build from scratch.
- GitHub: `silburt/DeepMoon`.

### 13. Automated Lunar Crater Identification with Chandrayaan-2 TMC-2 Images using Deep CNNs (Scientific Reports, 2024)
- Why relevant: A crater-detection U-Net specifically trained/validated on TMC-2 imagery (one of our three named sensors) — more directly applicable than DeepMoon (which uses DEMs, not optical TMC images) if we want an image-space (not elevation-space) crater/structure detector for the Fusion-detector-style filtering idea.

---

##  LOWER PRIORITY — Useful context, not immediately actionable

- PromptMID / MIFNet / S2M2-SAR — recent (2025) diffusion-model / foundation-model-based modality-invariant descriptor methods for optical-SAR matching. Interesting research direction (using vision foundation models like Stable Diffusion for modality-invariant features) but heavier and less proven than RIFT/RoMa/MatchAnything; worth a look only if simpler methods prove insufficient.
- SEN1-2 dataset & pseudo-Siamese SAR-optical matching (Hughes/Merkle et al.) — foundational but dated (2017–2018) work on learned SAR-optical patch matching; largely superseded by RIFT and the newer transformer-based methods above; useful mainly as historical/background context.
- Multi-Resolution SAR and Optical Remote Sensing Image Registration: A Review (arXiv:2502.01002) — a survey paper; good as a secondary reference/citation list (it's where RIFT and several other methods were cross-referenced from) rather than a primary technical source.

---

## Suggested next steps
Given what's been found, the highest-value next full-analysis targets (in priority order) are:
1. RIFT (illumination/radiation-invariant classical method — fills the biggest gap cheaply, no training data needed)
2. MatchAnything (learned, explicitly cross-modality, most direct attempt at exactly our unsolved problem)
3. RoMa (strongest illumination/viewpoint robustness evidence of any method found so far)
4. The two Chandrayaan-2 OHRC DEM papers together (as a "what does a working OHRC pipeline already look like" case study, distinct from a pure-matching-algorithm paper)

Say which one(s) you'd like analyzed in full P0–P6 format next.
