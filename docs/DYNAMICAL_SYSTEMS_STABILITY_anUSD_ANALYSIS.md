# anUSD Stability & Equilibrium Analysis
## A Piecewise-Smooth Dynamical Systems Framework

**Author:** Bonding Curves Research Group (BCRG)  
**Context:** Mathematical Foundations for Parameter Selection Under Uncertainty (PSUU)  
**Theoretical Reference:** Prof. Soumitro Banerjee, *Chaos, Fractals and Dynamical Systems* (NPTEL Course PL465CF583900B36A3)  
**Date:** November 10, 2026

---

## 1. Executive Summary & Theoretical Context

When analyzing anUSD across market cycles, standard linear financial models and spreadsheet calibrations fail because the mechanism is fundamentally a **non-linear, piecewise-smooth (PWS) hybrid dynamical system**. 

The system evolves under smooth, continuous stochastic vector fields during normal market trading, but undergoes **discontinuous jump state transitions (border collisions)** when the junior tranche value reaches predetermined reset boundaries ($H_u = 2.00$ and $H_d = 0.25$).

Drawing directly from the foundational dynamical systems framework established by Prof. Soumitro Banerjee (IISER Kolkata / IIT Kharagpur) in *Chaos, Fractals and Dynamical Systems*, this document formalizes:
1. The state-space representation of the anUSD dual-tranche vault.
2. The switching manifolds ($\Sigma_u, \Sigma_d$) and the **Saltation Matrix** governing boundary resets.
3. The avoidance of **Border-Collision Bifurcations (BCB)** and **Boundary Crises** (sudden destruction of stable attractors).
4. The four concrete mathematical stability criteria that our Parameter Selection Under Uncertainty (PSUU) pipeline aims to satisfy.

---

## 2. State-Space Formulation of the Vault

### 2.1 State Vector

The state of the anUSD collateral vault at any continuous time $t$ is represented by the state vector $\mathbf{x}(t) \in \mathbb{R}^4$:

$$
\mathbf{x}(t) = \begin{bmatrix} S(t) \\ V_A(t) \\ V_B(t) \\ \lambda(t) \end{bmatrix}
$$

Where:
* $S(t) \in \mathbb{R}^+$: The price of the underlying collateral asset (AVAX index), normalized to $S(0) = 1.00$.
* $V_A(t) \in \mathbb{R}^+$: The senior contractual debt liability (anBOND par claim), accruing coupon at contractual rate $R$.
* $V_B(t) \in \mathbb{R}$: The junior equity tranche value, absorbing residual collateral volatility:
  $$V_B(t) = 2 S(t) - V_A(t)$$
* $\lambda(t) \in \mathbb{R}^+$: The effective leverage multiplier of the junior tranche:
  $$\lambda(t) = \frac{2 S(t)}{V_B(t)} = \frac{2 S(t)}{2 S(t) - V_A(t)}$$

The nominal target operating equilibrium is given by:

$$
\mathbf{x}^* = \begin{bmatrix} 1.00 \\ 1.00 \\ 1.00 \\ 2.00 \end{bmatrix}
$$

---

## 3. The Switching Manifolds & Piecewise Vector Fields

The state space is partitioned into three distinct topological regions by two codimension-1 switching surfaces (borders):

```
State Space Partitioning:
─────────────────────────────────────────────────────────────────────────────
Region R_upper:      V_B >= H_u (2.00)   ──> Upward Reset (Forward Split 2:1)
─────────────────────────────────────────────────────────────────────────────
Switching Border Σ_u:  V_B = H_u = 2.00
─────────────────────────────────────────────────────────────────────────────
Smooth Domain D:     H_d < V_B < H_u     ──> Continuous Kou Jump-Diffusion
─────────────────────────────────────────────────────────────────────────────
Switching Border Σ_d:  V_B = H_d = 0.25
─────────────────────────────────────────────────────────────────────────────
Region R_lower:      V_B <= H_d (0.25)   ──> Downward Reset (Reverse Split 4:1)
─────────────────────────────────────────────────────────────────────────────
```

### 3.1 Smooth Interior Domain ($\mathcal{D}$)

Inside the interior domain $\mathcal{D} = \{ \mathbf{x} \mid H_d < V_B(t) < H_u \}$, the system evolves under smooth continuous dynamics driven by the stochastic price vector field $\mathbf{f}(\mathbf{x})$:

$$
d S(t) = \mu S(t) dt + \sigma S(t) dW(t) + S(t-) dJ(t)
$$

$$
d V_A(t) = R \cdot V_A(t) dt
$$

$$
d V_B(t) = 2 \, dS(t) - dV_A(t)
$$

Where $W(t)$ is standard Brownian motion, and $J(t)$ is a compound Poisson jump process with log-double-exponential jump amplitude (Kou jump-diffusion).

---

## 4. Discontinuous Jumps & The Saltation Matrix

In classical smooth systems (Banerjee, Lecture 2), local stability is analyzed via the Jacobian $\mathbf{J} = \frac{\partial \mathbf{f}}{\partial \mathbf{x}}$. However, in non-smooth systems with impact or reset boundaries (Banerjee, Lecture 37 & 39), the trajectory undergoes a discontinuous jump upon encountering the switching manifold $\Sigma$.

### 4.1 The Downward Reset Saltation Mapping

When the trajectory collides with the lower border $\Sigma_d = \{ \mathbf{x} \mid h_d(\mathbf{x}) = V_B - H_d = 0 \}$:
1. **Debt Amortization:** Senior claims receive $\gamma = 0.75$ in physical collateral.
2. **Liabilities Extinction:** Outstanding senior debt contracts: $V_A \to V_A - 0.75 = 0.25$.
3. **Collateral Contraction:** Remaining vault collateral: $2S \to 2S - 0.75 = 0.50$.
4. **Share Consolidation:** A 4-for-1 reverse split ($k_{\text{rev}} = 4$) renormalizes par value to $1.00$.

The transition across the discontinuity boundary is governed by the **Saltation Matrix** $\mathbf{S}_{\text{salt}}$ (Banerjee, Lecture 39):

$$
\mathbf{S}_{\text{salt}} = \mathbf{I} + \frac{(\mathbf{f}_+ - \mathbf{f}_-) \mathbf{n}^T}{\mathbf{n}^T \mathbf{f}_- + \frac{\partial h_d}{\partial t}}
$$

Where:
* $\mathbf{n} = \nabla h_d(\mathbf{x}) = \begin{bmatrix} 2 & -1 & 0 & 0 \end{bmatrix}^T$ is the normal vector to the switching manifold.
* $\mathbf{f}_-$ is the approaching vector field velocity just prior to reset.
* $\mathbf{f}_+$ is the departing vector field velocity immediately post-reset.

The complete cycle stability over an orbital period $\tau$ is determined by the **Monodromy Matrix** $\mathbf{M}$:

$$
\mathbf{M} = \mathbf{\Phi}(\tau, t_{\text{reset}}^+) \cdot \mathbf{S}_{\text{salt}} \cdot \mathbf{\Phi}(t_{\text{reset}}^-, 0)
$$

**Stability Condition:** For orbital stability around target leverage $\lambda^* = 2.0$, all Floquet multipliers (eigenvalues $\mu_i$ of $\mathbf{M}$) must reside strictly within the unit circle:

$$
\max_i |\mu_i(\mathbf{M})| < 1
$$

---

## 5. Non-Smooth Bifurcations & Boundary Crises

Prof. Banerjee's lectures establish two critical phenomena that directly inform stablecoin safety:

### 5.1 Border-Collision Bifurcations (BCB) (Lectures 31–35)

In smooth systems, bifurcations occur when eigenvalues cross the unit circle continuously. In piecewise-smooth maps, **Border-Collision Bifurcations** occur when an orbit collides with the border $\Sigma$. 

Because the piecewise Jacobians $\mathbf{J}_L$ (left of border) and $\mathbf{J}_R$ (right of border) differ discontinuously:
* An equilibrium point can transition from stable to unstable **in a single step** without intermediate continuous degradation.
* A poorly tuned reset ratio ($\gamma$) or boundary ($H_d$) can trigger a border collision that produces **chattering** (rapid successive resets in adjacent blocks) or chaotic burst volatility.

### 5.2 Dangerous Bifurcations & Boundary Crises (Lectures 21 & 36)

A stable attractor $\mathcal{A}$ operates within a defined **Basin of Attraction** $\mathcal{B}(\mathcal{A})$—the set of all initial market states that converge to stable equilibrium.

The boundary of this basin, $\partial \mathcal{B}(\mathcal{A})$, is formed by the stable manifold of an unstable saddle orbit, $W^s(\mathbf{x}_{\text{saddle}})$.

* **The Boundary Crisis:** When market volatility or excessive leverage pushes the attractor to collide with its basin boundary:
  $$\mathcal{A} \cap \partial \mathcal{B}(\mathcal{A}) \ne \emptyset$$
* **The Systemic Consequence:** The attractor is instantaneously **destroyed**. Trajectories escape the basin permanently. In financial terms, this represents an unrecoverable **liquidation death spiral / insolvency run**.

```
Basin of Attraction & Boundary Crisis Schematic:
─────────────────────────────────────────────────────────────────────────────
              [ Basin of Attraction B(A) ]
   ┌────────────────────────────────────────────────────────┐
   │                                                        │
   │           Target Attractor A                           │
   │           (λ* = 2.0x, P* = $1.00)                      │
   │                   •                                    │
   │                                                        │
   │                                     Basin Boundary     │
   │                                     ∂B(A) = W^s(x_saddle)
   └────────────────────────────────────────────┬───────────┘
                                                │
   Collision: A touches ∂B(A)  ═════════════════╪════════════> [ BOUNDARY CRISIS ]
                                                │               Attractor Destroyed
                                                ▼               (Death Spiral / Insolvency)
                                      [ Unstable Escape ]
─────────────────────────────────────────────────────────────────────────────
```

---

## 6. What We Wish to Achieve with the anUSD Design

Grounding our mechanism design in Prof. Banerjee's dynamical systems framework establishes four concrete mathematical objectives:

### Objective 1: Contraction of the Reset Map (Global Inward Projection)
The reset operator $g_{\text{reset}}: \Sigma_d \to \mathcal{D}$ must be strictly contractive relative to the target equilibrium $\mathbf{x}^*$:

$$
\| g_{\text{reset}}(\mathbf{x}) - \mathbf{x}^* \|_2 < \| \mathbf{x} - \mathbf{x}^* \|_2 \quad \forall \mathbf{x} \in \Sigma_d
$$

This ensures that whenever the market forces the vault to the lower barrier $H_d$, the reset maps the system **inward into the deep interior of the stable basin**, restoring 200% collateral coverage and 2.0× leverage, preventing chattering.

### Objective 2: Floquet Multiplier Confinement ($|\mu_i| < 1$)
The eigenvalues of the Monodromy Matrix must satisfy:

$$
\rho(\mathbf{M}) = \max_i |\mu_i(\mathbf{M})| \le 1 - \epsilon \quad (\epsilon > 0)
$$

ensuring exponential decay of perturbations following a reset event.

### Objective 3: Boundary Crisis Immunity across Historical Volatility
The distance between the operating attractor $\mathcal{A}$ and the basin boundary $\partial \mathcal{B}$ must strictly exceed the maximum empirical price shock:

$$
\text{dist}\left( \mathcal{A}, \; \partial \mathcal{B}(\mathcal{A}) \right) > \sup_{t} |\Delta S(t)|
$$

Under Theorem 1, anUSD guarantees invariance up to an instantaneous single-block price shock of **−75% from par** and **−60% from the reset barrier**, proving that black-swan price drops do not cause a boundary crisis.

### Objective 4: Elimination of Secondary Market Sinking (DEX Coupling)
A critical unmodeled frontier is the coupling between the reset payout (physical sAVAX) and secondary DEX liquidity. If senior holders liquidate sAVAX on DEXs post-reset, secondary slippage could push $S(t)$ further down, generating a secondary collision with $\Sigma_d$. 

The PSUU optimization must size the debt amortization fraction $\gamma$ and the reserve buffer $\beta$ such that:

$$
\left| \frac{\partial S}{\partial \text{DEX Sell}} \cdot \gamma \cdot V_A \right| < H_u - H_d
$$

preventing secondary border collisions.

---

## 7. Implications for the PSUU Optimization Pipeline

This dynamical systems formulation provides the rigorous theoretical justification for why **Parameter Selection Under Uncertainty (PSUU)** is essential:

1. **Parameter Vector Coupling:** The vector $\theta = (H_u, H_d, R, R', \gamma, \vec{\omega}, \Delta t)$ controls the position of the switching surfaces and the spectrum of the saltation matrix $\mathbf{S}_{\text{salt}}$.
2. **Non-Linear Bifurcation Boundaries:** The stability boundary is not a simple rectangle in parameter space; it is a complex, curved hypersurface defined by the border-collision bifurcation manifold.
3. **Simulation Rigor:** Only high-dimensional stochastic simulation (incorporating Kou jumps, secondary DEX depth, and unstaking queues) can map this hypersurface and identify the robust parameter cluster that guarantees asymptotic stability.

***

*Bonding Curves Research Group (BCRG)*
