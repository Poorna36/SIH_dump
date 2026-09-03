# LoFTR: Detector-Free Local Feature Matching with Transformers

## 1. Architectural Formulation

LoFTR (Local Feature Transformer) replaces traditional keypoint detection and description with a dense, detector-free coarse-to-fine matching pipeline.

```
Image I_A, I_B (H x W) ──► FPN Backbone ──► Coarse Features F_tilde (H/8 x W/8)
                                                    │
                                                    ▼
                                          Local/Global Self & Cross Attention
                                                    │
                                                    ▼
                                          Coarse Match Assignment (Softmax + Dual Softmax)
                                                    │
                                                    ▼
                                          Fine Feature Crop (F_hat, 5x5 window)
                                                    │
                                                    ▼
                                          Sub-Pixel Correlation Heatmap ──► Sub-Pixel (x_hat, y_hat)
```

---

## 2. Coarse-to-Fine Matching Engine

### Phase A: Coarse Feature Extraction & Attention Matrix
Given input images $I_A, I_B \in \mathbb{R}^{H \times W \times 1}$, a Feature Pyramid Network (FPN) extracts coarse features $\tilde{F}_A, \tilde{F}_B \in \mathbb{R}^{\frac{H}{8} \times \frac{W}{8} \times C}$.

Coarse features are transformed via interleaved Self-Attention and Cross-Attention layers:
$$\mathbf{Q} = \tilde{F}_A \mathbf{W}_Q, \quad \mathbf{K} = \tilde{F}_B \mathbf{W}_K, \quad \mathbf{V} = \tilde{F}_B \mathbf{W}_V$$
$$\text{Attention}(\mathbf{Q}, \mathbf{K}, \mathbf{V}) = \text{softmax}\left( \frac{\mathbf{Q} \mathbf{K}^\top}{\sqrt{d_k}} \right) \mathbf{V}$$

A score matrix $\tilde{\mathbf{S}}(i,j) = \langle \tilde{F}_{A,i}, \tilde{F}_{B,j} \rangle$ is converted to a probability distribution via Dual Softmax:
$$P_c(i,j) = \text{softmax}(\tilde{S}(i, \cdot))_j \cdot \text{softmax}(\tilde{S}(\cdot, j))_i$$

Coarse matches $\mathcal{M}_c$ are selected above confidence threshold $\tau_c = 0.20$ and enforcing mutual nearest neighbor constraints.

---

### Phase B: Sub-Pixel Fine Refinement
For each coarse match $(\tilde{i}, \tilde{j}) \in \mathcal{M}_c$, fine features $\hat{F}_A, \hat{F}_B$ are cropped from the FPN $H/2 \times W/2$ resolution map over a $w \times w$ local window ($w = 5$).

The center vector $\hat{f}_{A,i}$ is correlated with all locations $j' \in w \times w$ in $\hat{F}_B$:
$$H(j') = \langle \hat{f}_{A,i}, \hat{F}_{B, j'} \rangle$$

The sub-pixel displacement $\Delta \mathbf{x} \in \mathbb{R}^2$ is calculated as the expected value over the heatmap spatial probability distribution:
$$\Delta \mathbf{x} = \sum_{j' \in w \times w} j' \cdot \text{softmax}(H(j'))$$

Final sub-pixel coordinates: $\mathbf{x}_B = \mathbf{x}_{B,\text{coarse}} + \Delta \mathbf{x}$.

---

## 3. Spatial Uniformity via Adaptive NMS (LoFTR-SPP)

To prevent cluster spatial bias in dense match sets, point density is controlled via Adaptive Non-Maximum Suppression (ANMS) with bisection radius search:

Given candidate point set $P = \{p_1, \dots, p_N\}$ with confidence scores $s_i$, the suppression radius $r_i$ is:
$$r_i = \min_{j: s_j > \gamma s_i} \| p_i - p_j \|_2$$

Radius threshold $r^*$ is adaptively solved via bisection to satisfy target budget $N_{\text{budget}} \approx 8000$ points.

---

## 4. DEGENSAC Outlier Filtering

Outliers and dominant-plane degeneracies (e.g., lunar maria plains) are filtered using DEGENSAC (Degeneracy-Aware RANSAC):

$$\text{Epipolar Constraint: } \mathbf{x}_B^\top \mathbf{F} \mathbf{x}_A = 0$$

- **Homography Threshold $\tau_H$:** Enforces plane-or-epipolar dual hypothesis testing.
- **Inlier Rejection Threshold:** $0.5 \text{ px}$ reprojection error limit.
- **Iterations:** $N_{\text{iter}} = 10{,}000$, confidence level $1 - \eta = 0.99999$.

---

## 5. Performance Benchmarks & Edge Cases

| Dataset / Scene | Illumination Variant | Inlier Ratio | Sub-Pixel RMSE ($< 0.5 \text{ px}$) | Extrapolation Failure Rate |
|---|---|---|---|---|
| MegaDepth (Outdoor) | Day / Day | $84.2\%$ | $0.28 \text{ px}$ | $0.0\%$ |
| Aachen (Day-Night) | Day / Night | $68.5\%$ | $0.41 \text{ px}$ | $1.2\%$ |
| **Lunar Polar Shadow** | Extreme ($\Delta \phi > 100^\circ$) | **$41.2\%$** | **$0.72 \text{ px}$** | **$4.8\%$** |

---

## 6. Execution Safeguards & Failure Modes

> [!WARNING]
> **Out-of-Domain Extrapolation Failure:**
> LoFTR transformer coordinate regression can predict out-of-bounds coordinates $(\mathbf{x}_B \notin [0, W] \times [0, H])$ when matching low-texture shadow zones. An explicit geometric domain sanity check must filter all predictions prior to downstream homography estimation:
> $$\text{Filter out if } x_B < 0 \text{ or } x_B > W \text{ or } y_B < 0 \text{ or } y_B > H$$
