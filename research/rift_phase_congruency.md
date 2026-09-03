# RIFT: Radiation-Invariant Feature Transform via Phase Congruency & Maximum Index Maps

## 1. Mathematical Formulation of Phase Congruency (PC)

Phase Congruency (PC) provides a frequency-domain model of feature perception that is invariant to variations in image illumination, contrast, and sensor modality.

Unlike spatial gradient operators $\nabla I(x,y)$, which are sensitive to absolute intensity shifts, Phase Congruency measures the local phase alignment across logarithmic Gabor filter scales:

$$PC(x,y) = \frac{\sum_{o} \sum_{n} W_o(x,y) \lfloor A_{no}(x,y) \Delta \Phi_{no}(x,y) - T_o \rfloor}{\sum_{o} \sum_{n} A_{no}(x,y) + \epsilon}$$

where:
- $A_{no}(x,y)$ is the amplitude at scale $n$ and orientation $o$.
- $\Delta \Phi_{no}(x,y) = \cos(\phi_{no}(x,y) - \bar{\phi}_o(x,y))$ is the phase deviation from the mean phase $\bar{\phi}_o(x,y)$.
- $W_o(x,y)$ is a frequency spread weighting factor.
- $T_o$ is a noise threshold.

---

## 2. Maximum Index Map (MIM) Feature Descriptor

To create a discriminative descriptor resistant to non-linear radiometric distortions (NRD), the Maximum Index Map (MIM) is constructed from the response of a log-Gabor filter bank over $N_o$ orientations ($N_o = 6$ default):

```
Image I(x,y) ──► Log-Gabor Filters (Ns=4, No=6) ──► Orientation Layers O_1, ..., O_6
                                                            │
                                                            ▼
                                                MIM(x,y) = argmax_o (O_o(x,y))
                                                            │
                                                            ▼
                                                Log-Polar Grid Histogram Encoding
```

For each pixel $(x,y)$, the MIM value records the orientation index $o^*$ yielding the maximum response:
$$\text{MIM}(x,y) = \arg\max_{o \in \{1, \dots, N_o\}} \left( e_{no}(x,y)^2 + o_{no}(x,y)^2 \right)$$

where $e_{no}$ and $o_{no}$ are the even-symmetric and odd-symmetric log-Gabor filter outputs.

---

## 3. Algorithmic Pipeline & Rotation Invariance

```
  Input Image I_1, I_2
        │
        ▼
  Log-Gabor Filter Bank ──► Compute Phase Congruency Map PC(x,y)
        │
        ▼
  Moment Analysis ──► Detect Keypoints via Minimum & Maximum Moments
        │
        ▼
  MIM Construction ──► Construct Maximum Index Map for NxN Keypoint Patches
        │
        ▼
  Multi-MIM Rotation Search ──► Generate No Permuted Descriptors per Keypoint
        │
        ▼
  Distance Metric Matching ──► Normalized Euclidean Distance + Outlier Rejection
```

### Rotation Invariance Formulation
Because dominant orientation estimation fails under multi-modal contrast inversion, rotation invariance is achieved by constructing $N_o$ cyclic permutations of the MIM descriptor histogram for each keypoint:
$$\mathbf{H}_{\text{MIM}}^{(k)} = \text{Shift}\left( \mathbf{H}_{\text{MIM}}, k \right) \quad \text{for } k \in \{0, 1, \dots, N_o - 1\}$$

Matching performs minimum distance search over all $N_o$ rotated candidate representations.

---

## 4. Empirical Performance & Comparative Analysis

Benchmarked across 6 Non-Linear Radiometric Distortion (NRD) multi-sensor satellite datasets:

| Method | Feature Type | Scale Invariance | Success Rate (SR) | Mean Reprojection Error |
|---|---|---|---|---|
| SIFT | Gradient Histogram | Yes | $31.7\%$ | $3.84 \text{ px}$ |
| SAR-SIFT | Modified Gradient | Partial | $28.3\%$ | $4.12 \text{ px}$ |
| HOPC | Phase Template | No (Needs metadata) | $54.2\%$ | $2.81 \text{ px}$ |
| **RIFT** | **Phase Congruency + MIM** | **No (Pre-sampled GSD)** | **$100.0\%$** | **$1.85 \text{ px}$** |

---

## 5. Implementation Constraints & Scale Gap

> [!IMPORTANT]
> **Scale Space Extension Requirement:**
> RIFT natively lacks scale invariance and requires pre-resampling images to matched GSD resolution. For cross-sensor matching (e.g., OHRC $0.3\text{ m/px}$ vs. TMC-2 $5\text{ m/px}$), a Log-Gabor scale-space pyramid must envelope the RIFT detector:
> $$\sigma_s = \sigma_0 \cdot \gamma^s \quad \text{for } s \in \{0, 1, \dots, S-1\}$$
