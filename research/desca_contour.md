# DESCA: Differential Evolution Sample Consensus Algorithm for Outlier Rejection

## 1. Mathematical Formulation of Optimization Objective

DESCA replaces standard RANSAC random trial sampling with Differential Evolution (DE) optimization to maximize the count of geometrically consistent inlier matches under an affine transformation model.

Given match point pairs $(x_i, y_i) \leftrightarrow (x'_i, y'_i)$, the affine transformation model vector $\mathbf{X} \in \mathbb{R}^6$ is defined as:
$$\begin{bmatrix} x' \\ y' \end{bmatrix} = \begin{bmatrix} a_{11} & a_{12} \\ a_{21} & a_{22} \end{bmatrix} \begin{bmatrix} x \\ y \end{bmatrix} + \begin{bmatrix} t_x \\ t_y \end{bmatrix} \implies \mathbf{X} = [a_{11}, a_{12}, a_{21}, a_{22}, t_x, t_y]^\top$$

---

## 2. Differential Evolution Optimization Loop

```
  Two-Tier Match Extraction:
  - Clean Set S_clean (NNDR = 0.70)  ──► Leave-One-Out RMSE Initializer ──► Population X_0
  - Full Set S_full  (NNDR = 1.00)  ──► Fitness Evaluation Pool

  DE Loop (Generations G=200, F=0.9, Cr=0.9):
  1. Mutation:     V_i = X_r1 + F * (X_r2 - X_r3)
  2. Crossover:    U_{i,j} = V_{i,j} if rand() <= Cr else X_{i,j}
  3. Fitness:      Fitness(U_i) = Count( Inliers(U_i, S_full, tau_dist=1.0px) )
  4. Selection:    X_{i,g+1} = U_i if Fitness(U_i) > Fitness(X_{i,g}) else X_{i,g}
```

### Fitness Function Formulation
For candidate parameter vector $\mathbf{X}_k$:
$$f(\mathbf{X}_k) = \sum_{i \in \mathcal{S}_{\text{full}}} \mathbb{I}\left( \left\| \begin{bmatrix} x'_i \\ y'_i \end{bmatrix} - \mathbf{M}(\mathbf{X}_k) \begin{bmatrix} x_i \\ y_i \\ 1 \end{bmatrix} \right\|_2 \le \tau_{\text{dist}} \right)$$

where $\tau_{\text{dist}} = 1.0 \text{ px}$ reprojection error threshold.

---

## 3. Data-Driven Population Initialization

> [!CRITICAL]
> **Random Initialization Failure:**
> Random initialization of vector $\mathbf{X}$ causes DE optimization to diverge into local minima, resulting in $0$ inlier matches. Initialization requires seeding via Leave-One-Out RMSE minimization over $\mathcal{S}_{\text{clean}}$:

$$\mathbf{X}_{\text{seed}} = \arg\min_{\mathbf{X}} \sum_{j \in \mathcal{S}_{\text{clean}} \setminus \{i\}} \left\| \mathbf{x}'_j - \mathbf{M}(\mathbf{X}) \mathbf{x}_j \right\|_2^2$$

---

## 4. Empirical Performance Benchmarks

Evaluated against classical consensus algorithms over optical satellite remote sensing datasets:

| Outlier Rejection Engine | Initialization Model | Mean Inlier Yield (NCMP) | Sub-Pixel RMSE | Execution Time per 1k Points |
|---|---|---|---|---|
| RANSAC | Random 3-point sample | $84$ | $1.42 \text{ px}$ | **$12 \text{ ms}$** |
| FSC (Fast Sample Consensus) | Top 10% ratio matches | $131$ | $1.08 \text{ px}$ | $18 \text{ ms}$ |
| PSOSAC (Particle Swarm) | Random PSO swarm | $92$ | $1.25 \text{ px}$ | $210 \text{ ms}$ |
| **DESCA** | **Two-Tier LOO RMSE** | **$168$** | **$0.78 \text{ px}$** | $145 \text{ ms}$ |

---

## 5. Failure Modes & Limitations

- **Local Relief Failure:** Global 6-DOF affine models cannot accommodate non-planar topographic relief (e.g., steep crater walls).
- **Required Mitigation:** Upgrade transformation model vector $\mathbf{X}$ from 6-DOF affine to an 8-DOF Homography or 12-DOF Biquadratic Polynomial matrix.
