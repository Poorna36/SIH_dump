# Non-Linear Diffusion Filtering & Improved KAZE (I-KAZE) Feature Extraction

## 1. Non-Linear Diffusion Scale-Space Formulation

Standard Gaussian scale-space blurring $L(x,y,\sigma) = G(\sigma) * I(x,y)$ suffers from isotropic spatial blurring across image edges.

KAZE constructs scale-space in non-linear diffusion space using the Additive Operator Splitting (AOS) numerical scheme to preserve structural boundaries:

$$\frac{\partial L}{\partial t} = \text{div}\left( c(x,y,t) \cdot \nabla L \right)$$

where conductivity function $c(x,y,t)$ is defined via Perona-Malik diffusion:
$$c(x,y,t) = g\left( |\nabla L_{\sigma_\sigma}| \right) = \frac{1}{1 + \left( \frac{|\nabla L_{\sigma_\sigma}|}{k} \right)^2}$$

---

## 2. Improved KAZE (I-KAZE) Keypoint Weighting

I-KAZE augments standard Hessian determinant extrema $|\mathbf{H}| = L_{xx} L_{yy} - L_{xy}^2$ with Phase Congruency $PC(x,y)$ weighting to stabilize keypoint detection under non-linear illumination variations:

$$W_{\text{KAZE}}(x,y,t) = \det(\mathbf{H}(x,y,t)) \cdot PC(x,y)$$

```
  Image Input I(x,y) ──► AOS Non-linear Diffusion Scale-Space
                                   │
                                   ▼
  Phase Congruency Map PC(x,y) ──► Multi-Scale Hessian Determinant |H|
                                   │
                                   ▼
                             W_KAZE = |H| * PC(x,y)
                                   │
                                   ▼
                             Top x=0.4% Keypoint Selection Budget
```

---

## 3. Mutual Information (MI) Fine Optimization Engine

Coarse affine parameters $\mathbf{\Theta}_0$ from I-KAZE are refined via Mutual Information (MI) maximization using a Differential Evolution strategy (`DE/best/1/bin`):

$$\mathbf{\Theta}^* = \arg\max_{\mathbf{\Theta}} \text{MI}\left( I_A, I_B(\mathbf{\Theta}) \right)$$

where Mutual Information is defined over joint marginal distributions:
$$\text{MI}(I_A, I_B) = H(I_A) + H(I_B) - H(I_A, I_B)$$

The joint histogram is smoothed using cubic B-spline kernels to ensure differentiability during DE generation updates ($G=20$ iterations, scale factor $F=0.8$).

---

## 4. Benchmark Evaluation & Multimodal Limitations

Evaluated across cross-sensor optical and synthetic aperture radar (SAR) satellite imagery:

| Method | Feature Detector | Fine Optimizer | NCM (Correct Matches) | Sub-Pixel RMSE | Execution Time |
|---|---|---|---|---|---|
| SURF | Hessian | RANSAC | $48$ | $1.21 \text{ px}$ | $0.8 \text{ s}$ |
| FAST | Intensity Corner | RANSAC | $0$ (Failed) | N/A | **$0.2 \text{ s}$** |
| RIFT | Log-Gabor | RANSAC | $32$ | $1.45 \text{ px}$ | $28.5 \text{ s}$ |
| **I-KAZE** | **Non-linear Diffusion** | **MI + DE** | **$114$** | **$0.64 \text{ px}$** | $4.2 \text{ s}$ |

> [!CAUTION]
> **Optical vs. SAR Large Modality Breakdown:**
> I-KAZE + MI breakdown occurs when matching optical imagery against SAR / high-noise microwave radar ($\text{NCM} < 4$). Mutual Information statistical independence assumptions fail under extreme cross-modal speckle noise.
