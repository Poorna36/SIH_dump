# Architecture & Documentation Session Logs
> **Repository:** Lunar Multi-Modal Image Correspondence Suite (SIH Problem Statement 26166)  
> **Documentation Directory:** `/ARCHITECTURE/docs`

---

## 📌 Session 01: Initial Documentation Restructuring & Consolidation

### **Date:** 2026-08-29
**Session Objective:** Establish a centralized, well-structured documentation repository under `/ARCHITECTURE/docs` consolidating all system specifications, research papers, pipeline formulations, and evaluation reviews into organized thematic pillars.

---

### 🏛️ Key Decisions & Structural Organization

1. **Standardized Directory Hierarchy:**
   - Established `/ARCHITECTURE/docs` with 5 thematic sub-directories to eliminate root clutter and provide clear separation of concerns:
     - `01_master_architecture/`: Master architecture specifications, reviews, design companion, and implementation details.
     - `02_pipeline_specifications/`: Execution pipelines, algorithmic flows, and end-to-end specifications.
     - `03_feature_matching_and_deep_learning/`: Deep learning (LoFTR, SuperGlue), frequency methods (RIFT Phase Congruency), non-linear scale space (KAZE), and comparative analyses.
     - `04_subpixel_refinement_and_geometry/`: Sub-pixel phase correlation, NASA Ames Stereo Pipeline techniques, topological crater graph matching (CNSFM), and radiometric normalization.
     - `05_research_and_benchmarks/`: Academic paper mappings, SIH 26166 reference tables, and external tooling/datasets.

2. **Master Navigation Hub (`README.md`):**
   - Created an interactive portal in [`/ARCHITECTURE/docs/README.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/README.md) with complete clickable links, end-to-end Mermaid pipeline diagrams, sensor comparison matrix, and quick reference links.

3. **Standardized Implementation Specs:**
   - Converted unformatted architecture notes (`impl_arch`) into [`01_master_architecture/IMPLEMENTATION_ARCHITECTURE.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/01_master_architecture/IMPLEMENTATION_ARCHITECTURE.md).

---

### 🗺️ Document Inventory & Cross-Reference Mapping

| Module / Pillar | Source File | Structured Path in `/ARCHITECTURE/docs/` |
| :--- | :--- | :--- |
| **Master Architecture** | `MASTER_SYSTEM_ARCHITECTURE.md` | [`01_master_architecture/MASTER_SYSTEM_ARCHITECTURE.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/01_master_architecture/MASTER_SYSTEM_ARCHITECTURE.md) |
| **Optimal Architecture** | `BEST_ARCHITECTURE_KNOWN.md` | [`01_master_architecture/BEST_ARCHITECTURE_KNOWN.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/01_master_architecture/BEST_ARCHITECTURE_KNOWN.md) |
| **Implementation Spec** | `impl_arch` | [`01_master_architecture/IMPLEMENTATION_ARCHITECTURE.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/01_master_architecture/IMPLEMENTATION_ARCHITECTURE.md) |
| **Architecture Companion** | `arch_companion.md` | [`01_master_architecture/ARCHITECTURE_COMPANION.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/01_master_architecture/ARCHITECTURE_COMPANION.md) |
| **Architecture Review** | `architecture_review.md` | [`01_master_architecture/ARCHITECTURE_REVIEW.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/01_master_architecture/ARCHITECTURE_REVIEW.md) |
| **Doc Review** | `architecture_doc_review.md` | [`01_master_architecture/ARCHITECTURE_DOC_REVIEW.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/01_master_architecture/ARCHITECTURE_DOC_REVIEW.md) |
| **Best Pipeline** | `BEST_PIPELINE.md` | [`02_pipeline_specifications/BEST_PIPELINE.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/02_pipeline_specifications/BEST_PIPELINE.md) |
| **Complete Pipeline** | `complete_pipeline.md` | [`02_pipeline_specifications/COMPLETE_PIPELINE.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/02_pipeline_specifications/COMPLETE_PIPELINE.md) |
| **Phase Congruency (RIFT)**| `RIFT_extracted.md` | [`03_feature_matching_and_deep_learning/RIFT_PHASE_CONGRUENCY.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/03_feature_matching_and_deep_learning/RIFT_PHASE_CONGRUENCY.md) |
| **SuperGlue GNN** | `SuperGlue.md` | [`03_feature_matching_and_deep_learning/SUPERGLUE_GNN.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/03_feature_matching_and_deep_learning/SUPERGLUE_GNN.md) |
| **LoFTR Transformer** | `LoFTR_IMC21.md` | [`03_feature_matching_and_deep_learning/LOFTR_DETECTOR_FREE.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/03_feature_matching_and_deep_learning/LOFTR_DETECTOR_FREE.md) |
| **KAZE Scale Space** | `KAZE(2026).md` | [`03_feature_matching_and_deep_learning/KAZE_NONLINEAR_SCALE_SPACE.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/03_feature_matching_and_deep_learning/KAZE_NONLINEAR_SCALE_SPACE.md) |
| **Classical vs DL Analysis**| `Traditional_vs_DeepLearning_FeatureMatching.md` | [`03_feature_matching_and_deep_learning/TRADITIONAL_VS_DEEP_LEARNING.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/03_feature_matching_and_deep_learning/TRADITIONAL_VS_DEEP_LEARNING.md) |
| **DESCA Descriptor** | `DESCA(Dr.Sourabh).md` | [`03_feature_matching_and_deep_learning/DESCA_DESCRIPTOR_ANALYSIS.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/03_feature_matching_and_deep_learning/DESCA_DESCRIPTOR_ANALYSIS.md) |
| **SIFT IIRS to WAC** | `SIFT-IIRS-WAC_extracted.md` | [`03_feature_matching_and_deep_learning/SIFT_IIRS_WAC_ANALYSIS.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/03_feature_matching_and_deep_learning/SIFT_IIRS_WAC_ANALYSIS.md) |
| **MoonMetaSync** | `MoonMetaSync_extracted.md` | [`03_feature_matching_and_deep_learning/MOONMETASYNC_ANALYSIS.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/03_feature_matching_and_deep_learning/MOONMETASYNC_ANALYSIS.md) |
| **Crater Matching (CNSFM)** | `CNSFM_Crater_Neighborhood_Matching.md` | [`04_subpixel_refinement_and_geometry/CNSFM_CRATER_NEIGHBORHOOD_MATCHING.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/04_subpixel_refinement_and_geometry/CNSFM_CRATER_NEIGHBORHOOD_MATCHING.md) |
| **Phase Correlation** | `HybridPhaseCorrelation(2026).md` | [`04_subpixel_refinement_and_geometry/HYBRID_PHASE_CORRELATION.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/04_subpixel_refinement_and_geometry/HYBRID_PHASE_CORRELATION.md) |
| **NASA SubPixel Refinement**| `NASA_SubPixel_Refinment(1d).md` | [`04_subpixel_refinement_and_geometry/NASA_SUBPIXEL_REFINEMENT.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/04_subpixel_refinement_and_geometry/NASA_SUBPIXEL_REFINEMENT.md) |
| **Radiometric Normalization**| `Radiometric_Normalization_Analysis.md` | [`04_subpixel_refinement_and_geometry/RADIOMETRIC_NORMALIZATION_ANALYSIS.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/04_subpixel_refinement_and_geometry/RADIOMETRIC_NORMALIZATION_ANALYSIS.md) |
| **Reference Mapping** | `ai_suggestions/SIH26166_reference_mapping.md` | [`05_research_and_benchmarks/SIH26166_REFERENCE_MAPPING.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/05_research_and_benchmarks/SIH26166_REFERENCE_MAPPING.md) |
| **Supplementary Research** | `ai_suggestions/SIH26166_supplementary_research(2).md` | [`05_research_and_benchmarks/SIH26166_SUPPLEMENTARY_RESEARCH.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/05_research_and_benchmarks/SIH26166_SUPPLEMENTARY_RESEARCH.md) |
| **Additional Resources** | `ai_suggestions/Additional_Resources_SIH26166.md` | [`05_research_and_benchmarks/ADDITIONAL_RESOURCES.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/05_research_and_benchmarks/ADDITIONAL_RESOURCES.md) |

---

### 🚀 Next Steps & Engineering Roadmap

1. **Pipeline Modular Implementation:**
   - Package the 6-stage cascade into reusable Python modules under `src/core/` (Preconditioning, Feature Extraction, Matching, Filtering, Refinement, Export).
2. **GPU Acceleration Benchmark:**
   - Benchmark Phase Congruency via CuPy / PyTorch vs OpenCV SIFT / KAZE CPU baseline.
3. **Automated Testing Suite:**
   - Prepare synthetic lunar test pairs with controlled rotation ($0-360^\circ$), scale factors ($1\times - 320\times$), and illumination variances ($0-80^\circ$ incidence shifts).
