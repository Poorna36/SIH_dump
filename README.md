# Lunar Multi-Modal Image Correspondence Suite
> **Smart India Hackathon (SIH) — Problem Statement 26166**

Autonomous, sub-pixel, multi-modal correspondence matching across lunar orbital imagery (Chandrayaan-2 **OHRC**, **TMC-2**, **IIRS**; LRO **NAC**, **WAC**; **SELENE TC/MI**; **LOLA** DEMs).

---

## ⚡ Quick Access

*   🏛️ **Master Architecture Guide**: [`BEST_ARCHITECTURE_KNOWN.md`](file:///Abhi/Projects/SIH/BEST_ARCHITECTURE_KNOWN.md)
*   🚀 **Optimal Pipeline Specification**: [`BEST_PIPELINE.md`](file:///Abhi/Projects/SIH/BEST_PIPELINE.md)
*   📚 **Complete Documentation Portal**: [`ARCHITECTURE/docs/README.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/README.md)
*   📝 **Documentation Session Logs**: [`ARCHITECTURE/docs/SESSION_LOGS.md`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs/SESSION_LOGS.md)

---

## 📂 Documentation Directory Structure

All architectural specifications, algorithmic breakdowns, research papers, and evaluations are located directly inside [`ARCHITECTURE/docs/`](file:///Abhi/Projects/SIH/ARCHITECTURE/docs):

```
ARCHITECTURE/
└── docs/                                      # Complete Documentation Hub
    ├── README.md                              # Master documentation portal & index
    ├── SESSION_LOGS.md                        # Documentation session log & decision tracker
    │
    ├── MASTER_SYSTEM_ARCHITECTURE.md          # End-to-end master system architecture
    ├── BEST_ARCHITECTURE_KNOWN.md             # Optimal architecture synthesis
    ├── IMPLEMENTATION_ARCHITECTURE.md         # 6-stage implementation architecture
    ├── ARCHITECTURE_COMPANION.md              # Rationale, tradeoffs & deep dives
    ├── ARCHITECTURE_REVIEW.md                 # Failure modes & peer review
    ├── ARCHITECTURE_DOC_REVIEW.md             # Design consistency review
    │
    ├── BEST_PIPELINE.md                       # Optimal end-to-end pipeline
    ├── COMPLETE_PIPELINE.md                   # Full pipeline specification
    │
    ├── RIFT_PHASE_CONGRUENCY.md               # Phase Congruency (RIFT)
    ├── SUPERGLUE_GNN.md                       # Graph Neural Network matching
    ├── LOFTR_DETECTOR_FREE.md                 # Detector-free Transformer matching
    ├── KAZE_NONLINEAR_SCALE_SPACE.md          # Non-linear scale space detection
    ├── TRADITIONAL_VS_DEEP_LEARNING.md        # Benchmarked comparative analysis
    ├── DESCA_DESCRIPTOR_ANALYSIS.md           # Illumination-invariant DESCA
    ├── SIFT_IIRS_WAC_ANALYSIS.md              # Transitive SIFT registration
    ├── MOONMETASYNC_ANALYSIS.md               # SPICE / metadata synchronization
    │
    ├── CNSFM_CRATER_NEIGHBORHOOD_MATCHING.md  # Topological crater graph matching
    ├── HYBRID_PHASE_CORRELATION.md            # Sub-pixel FFT phase correlation
    ├── NASA_SUBPIXEL_REFINEMENT.md            # NASA Ames Stereo Pipeline refinement
    ├── RADIOMETRIC_NORMALIZATION_ANALYSIS.md  # Wallis / CLAHE radiometric filtering
    │
    ├── SIH26166_REFERENCE_MAPPING.md          # Reference paper mapping
    ├── SIH26166_SUPPLEMENTARY_RESEARCH.md     # Extended sensor research
    └── ADDITIONAL_RESOURCES.md                # Datasets & ISIS3 tool guides
```
