# SuperGlue: Graph Neural Network Matching & Optimal Transport

## 1. Architectural Formulation

SuperGlue frames local feature matching as a graph assignment problem solved jointly via Graph Neural Networks (GNN) and a differentiable Sinkhorn Optimal Transport layer.

```
Keypoints A (x, y, c) ──► Positional Encoding ──┐
Descriptors d_A      ────────────────────────────┼──► Attentive GNN ──► Matching Descriptors f_A, f_B
Keypoints B (x, y, c) ──► Positional Encoding ──┤    (L=9 Layers)            │
Descriptors d_B      ────────────────────────────┘                            ▼
                                                                     Score Matrix S = f_A^T f_B
                                                                              │
                                                                              ▼
                                                                     Sinkhorn Optimal Transport (T=100)
                                                                              │
                                                                              ▼
                                                                     Assignment Matrix P (with Dustbins)
```

---

## 2. Attentive Graph Neural Network (GNN)

Given two sets of keypoints $A$ and $B$, keypoint positions $p = (x, y)$ are embedded with confidence scores $c$ using a Multi-Layer Perceptron (MLP):
$$\mathbf{x}_i^{(0)} = \mathbf{d}_i + \text{MLP}(p_i, c_i)$$

The GNN performs $L=9$ alternating layers of **Self-Attention** (intra-image keypoint context) and **Cross-Attention** (inter-image keypoint matching):

$$\mathbf{x}_i^{(\ell+1)} = \mathbf{x}_i^{(\ell)} + \text{MLP}\left( [\mathbf{x}_i^{(\ell)} \;\|\; \mathbf{m}_{\mathcal{E} \to i}] \right)$$

where the message $\mathbf{m}_{\mathcal{E} \to i}$ is aggregated over graph edges $\mathcal{E}$ via multi-head attention:
$$\mathbf{m}_{\mathcal{E} \to i} = \sum_{j \in \mathcal{E}} \text{softmax}_j \left( \frac{\mathbf{q}_i^\top \mathbf{k}_j}{\sqrt{d}} \right) \mathbf{v}_j$$

---

## 3. Differentiable Sinkhorn Optimal Transport

The refined matching descriptors $\mathbf{f}_A \in \mathbb{R}^{M \times D}$ and $\mathbf{f}_B \in \mathbb{R}^{N \times D}$ yield a pairwise assignment score matrix $\mathbf{S} \in \mathbb{R}^{M \times N}$:
$$S_{i,j} = \langle \mathbf{f}_{A,i}, \mathbf{f}_{B,j} \rangle$$

To accommodate unmatched keypoints due to shadows, occlusions, or out-of-frame boundaries, $\mathbf{S}$ is augmented with learnable **dustbin** parameters $\mathbf{z} \in \mathbb{R}$:

$$\bar{\mathbf{S}}_{i,j} = \begin{cases} S_{i,j} & \text{if } i \le M, j \le N \\ z & \text{if } i = M+1 \text{ or } j = N+1 \end{cases}$$

The assignment matrix $\mathbf{P} \in \mathbb{R}^{(M+1) \times (N+1)}$ is computed by Sinkhorn-Knopp iterations:
$$\mathbf{P} = \text{Sinkhorn}(\bar{\mathbf{S}})$$

satisfying the marginal probability constraints:
$$\mathbf{P} \mathbf{1}_{N+1} = \mathbf{a}, \quad \mathbf{P}^\top \mathbf{1}_{M+1} = \mathbf{b}$$

where $\mathbf{a} = [1, \dots, 1, N]^\top$ and $\mathbf{b} = [1, \dots, 1, M]^\top$.

---

## 4. Empirical Performance Benchmarks

Evaluated on Aachen Day-Night & MegaDepth benchmark datasets:

| Feature Engine | Matcher | Precision (P) | Recall (R) | Matching Score | AUC @ $5^\circ$ Pose Error | Memory Footprint |
|---|---|---|---|---|---|---|
| SIFT | Nearest Neighbor | $42.1\%$ | $38.4\%$ | $18.2\%$ | $24.6\%$ | Low |
| SIFT | SuperGlue | **$88.4\%$** | $72.1\%$ | $44.8\%$ | **$58.2\%$** | $320 \text{ MB}$ |
| SuperPoint | Nearest Neighbor | $51.2\%$ | $44.1\%$ | $26.5\%$ | $36.1\%$ | VRAM $1.2 \text{ GB}$ |
| **SuperPoint** | **SuperGlue** | **$94.3\%$** | **$81.6\%$** | **$62.3\%$** | **$73.8\%$** | **VRAM $1.8 \text{ GB}$** |

---

## 5. Execution Constraints & Operational Profile

- **Hardware Dependency:** GPU required for real-time GNN inference (TensorRT / PyTorch C++ API).
- **Execution Speed:** $\approx 15 \text{ ms}$ on NVIDIA RTX 3090 / T4 for 1024 keypoints per image pair.
- **Dustbin Thresholding:** Candidate matches are filtered via confidence threshold $\tau_p$:
  $$\text{Match Accepted if } P_{i,j} \ge \tau_p \quad (\tau_p = 0.20 \text{ default})$$
- **Known Failure Mode:** Severe keypoint dropouts in low-texture lunar maria; requires front-end detector repeatability.
