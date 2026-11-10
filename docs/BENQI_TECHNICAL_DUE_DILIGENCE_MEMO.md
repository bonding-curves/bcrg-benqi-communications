# anUSD Technical Due Diligence Memorandum
## Architectural Specifications, Formal Accounting Equations, and Stress Dynamics

**To:** BENQI Strategy & Risk Engineering Team  
**From:** Bonding Curves Research Group (BCRG)  
**Date:** November 10, 2026  
**Reference:** *anUSD Executive Proposal (October 2026)*  
**Status:** Technical Working Draft

---

## 1. Context & Scope

In October 2026, BCRG presented the *anUSD Executive Proposal* to BENQI leadership, introducing a liquid-staked native dollar engine designed to scale sAVAX TVL through four product channels while replacing liquidation auctions with internal vault accounting.

During subsequent technical discussions, four core architectural areas were identified for detailed due diligence:

1. **Tranche Mechanics & Dynamic Leverage:** The formal mint/redeem rules, collateralization ratios, and mathematical maintenance of 2.0x leverage across market cycles.
2. **Yield Transmission & Capital Allocation:** The precise waterfall routing consensus staking rewards between senior fixed claims, amplified yield tokens, and protocol surplus.
3. **Downside Stress Mechanics:** Vault behavior, internal debt amortization, and solvency boundaries during prolonged market contractions (e.g., a 50%–60% monthly decline) and discontinuous flash crashes.
4. **Comparative Mechanism Architecture:** Structural risk trade-offs relative to debt-auction (MakerDAO/Sky), stability-pool (Liquity), and algorithmic (Terra) paradigms.

This memorandum provides the balance-sheet accounting, formal reset mechanics, and stress-testing specifications addressing these areas, and identifies the open modeling frontiers that require further research.

---

## 2. Vault Architecture & Formal Accounting Equations

### 2.1 Collateral Decomposition & Balance Sheet Equations

Rather than issuing debt against volatile collateral through an isolated Collateralized Debt Position (CDP) subject to third-party liquidation auctions, anUSD pools deposited sAVAX and partitions the aggregate claims into two complementary tranches against the underlying reserve:

```
[ sAVAX Collateral Deposited in Vault: $200 ]
                     │
       ┌─────────────┴─────────────┐
       ▼                           ▼
[ Senior Claim: anBOND ]    [ Junior Equity: anAVAX ]
         $100                        $100
  • Capital preservation      • Subordinated first-loss
  • Fixed coupon claim        • Absorbs collateral volatility
    (e.g., R = 7.3% p.a.)      • Exactly 2.0x initial leverage
                                ($100 equity controls $200 collateral)
```

The underlying mechanics are defined by four foundational balance-sheet equations:

1. **Total Collateral Value ($C_t$):**  
   Let $S_t = P_{\text{AVAX}}(t) / P_{\text{AVAX}}(0)$ denote the normalized AVAX spot price index ($S_0 = 1.00$). For an initial vault deposit of $2.00 of sAVAX:
   $$C_t = 2.00 \cdot S_t$$

2. **Senior Contractual Debt ($V_A(t)$):**  
   The senior tranche (anBOND) represents a preferred debt claim that accrues at annual coupon rate $R$:
   $$V_A(t) = 1.00 \cdot (1 + R \cdot t)$$
   At epoch inception ($t = 0$), $V_A(0) = \$1.00$.

3. **Junior Equity Value ($V_B(t)$):**  
   The junior tranche (anAVAX) represents the residual collateral value after senior debt obligations are accounted for:
   $$V_B(t) = C_t - V_A(t) = 2.00 \cdot S_t - V_A(t)$$

4. **Dynamic Junior Leverage ($\lambda_t$):**  
   Effective junior leverage is defined as the ratio of gross collateral to junior equity:
   $$\lambda_t = \frac{C_t}{V_B(t)} = \frac{2.00 \cdot S_t}{2.00 \cdot S_t - V_A(t)}$$
   At inception ($S_0 = 1.00, V_A(0) = \$1.00$):
   $$\lambda_0 = \frac{2.00}{2.00 - 1.00} = \mathbf{2.0\times}$$

---

### 2.2 Secondary Tranche Subordination: anUSD and anYIELD

Because anBOND accrues coupon value over time, its terminal redemption value drifts above par (e.g., $1.01, $1.02). To establish a stable, unit-pegged currency suitable for payments and treasury operations, anBOND is partitioned into a secondary tranche pair:

```
                 [ 2 Units of anBOND ($2.00 par) ]
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
       [ anUSD Stablecoin ]           [ anYIELD High-Yield ]
             $1.00 par                      $1.00 par
       • Non-volatile $1.00 peg       • Concentrated residual coupon
       • Optional base rate R'        • Yield formula: R_anYIELD = 2R - R'
         (e.g., 2.5% p.a.)
```

The secondary conversion rules enforce two conservation laws:

* **Par Conservation Rule:**
  $$P_{\text{anUSD}} + P_{\text{anYIELD}} = 2 \cdot P_{\text{anBOND}} = \$2.00$$
* **Coupon Transmission Rule:**
  $$R_{\text{anYIELD}} = 2R - R'$$
  At baseline reference rates ($R = 7.3\%$ and $R' = 2.5\%$ p.a.), the resulting yield on anYIELD evaluates to:
  $$R_{\text{anYIELD}} = 2(7.3\%) - 2.5\% = \mathbf{12.1\% \text{ p.a.}}$$

**Minting and Redemption Conversion Rules:**
* **Issuance (Mint):** Locking **2 units of anBOND** mints **1 unit of anUSD + 1 unit of anYIELD**.
* **Settlement (Redeem):** Surrendering **1 unit of anUSD + 1 unit of anYIELD** redeems **2 units of anBOND**.

*(Accounting Note: Final technical specifications must confirm whether the 2:1 ratio is enforced strictly in unit quantities or dollar-equivalent nominal value, ensuring strict value conservation across multi-epoch operations).*

---

### 2.3 Dynamic Leverage Evolution Across Price Trajectories

Leverage in the junior tranche is dynamic; it responds continuously to underlying collateral value:

$$\lambda_t = \frac{C_t}{V_B(t)}$$

Assuming an initial state of $2.00 in sAVAX collateral supporting $1.00 of senior debt and $1.00 of junior equity:

```
   AVAX Drops -37.5%                      Initial State                        AVAX Rises +50%
─────────────────────────────────      ───────────────────────      ─────────────────────────────────
• Collateral:          $1.25           • Collateral: $2.00          • Collateral:          $3.00
• Senior Claim:        $1.00           • Senior Claim: $1.00        • Senior Claim:        $1.00
• Junior Equity:       $0.25           • Junior Equity: $1.00       • Junior Equity:       $2.00
─────────────────────────────────      ───────────────────────      ─────────────────────────────────
▶ LEVERAGE RISES TO:   5.0x            ▶ LEVERAGE IS: 2.0x          ▶ LEVERAGE DECAYS TO:  1.5x
  ($1.25 / $0.25)                        ($2.00 / $1.00)              ($3.00 / $2.00)
  [Triggers Downward Reset]                                           [Triggers Upward Reset]
```

* **When AVAX Rises +50% ($S_t = 1.50$):** Collateral value expands to $3.00. The senior claim remains $1.00, increasing junior equity to $2.00. Effective leverage naturally compresses:
  $$\lambda = \frac{\$3.00}{\$2.00} = \mathbf{1.5\times}$$
* **When AVAX Drops −37.5% ($S_t = 0.625$):** Collateral contracts to $1.25. Subtracting the $1.00 senior claim leaves $0.25 in junior equity. Effective leverage expands:
  $$\lambda = \frac{\$1.25}{\$0.25} = \mathbf{5.0\times}$$

To bound leverage expansion and protect senior solvency without liquidation auctions, the protocol executes automated, deterministic balance-sheet resets at predefined price barriers.

---

### 2.4 Internal Reset & Rebalancing Mechanics (The Share Splits)

The October 2026 Executive Brief noted that anUSD provides contagion protection because *"shares consolidate internally"* during severe drawdowns. Below are the formal balance-sheet mechanics:

#### 1. Upward Reset (Triggered at $V_B \ge H_u = \$2.00$ / +50% AVAX Gain)
* **Objective:** Realize accumulated equity gains and restore the junior tranche to target 2.0× leverage.
* **Mechanism:** 
  1. The protocol crystallizes junior capital gains of $\$1.00$ per share ($\Delta \Pi = V_B - 1.00 = \$1.00$) and distributes it directly to junior holders in physical sAVAX.
  2. The contract executes a **2-for-1 forward share split**:
     $$N_{\text{new}} = 2 \cdot N_{\text{old}}, \quad V_B^{\text{new}} = \$1.00$$
* **Result:** System leverage cleanly recalibrates to $\lambda = \frac{\$2.00}{\$1.00} = \mathbf{2.0\times}$.

#### 2. Downward Reset (Triggered at $V_B \le H_d = \$0.25$ / −37.5% AVAX Drop)
* **Objective:** Eliminate junior leverage blowout, amortize senior liabilities, and restore balance-sheet collateralization prior to any senior capital impairment.
* **Mechanism:**
  1. **Senior Debt Amortization:** The contract automatically pays senior holders **$0.75 of their $1.00 claim directly in physical sAVAX**, extinguishing 75% of protocol liabilities:
     $$V_A^{\text{rem}} = \$1.00 - \$0.75 = \$0.25$$
     $$C^{\text{rem}} = \$1.25 - \$0.75 = \$0.50$$
  2. **Capital Consolidation (4-for-1 Reverse Split):** The remaining $0.25 senior claim and $0.25 junior equity claim merge via an internal 4-for-1 reverse split ($k = 4$):
     $$N_{\text{new}} = \frac{N_{\text{old}}}{4}, \quad V_A^{\text{new}} = \$1.00, \quad V_B^{\text{new}} = \$1.00$$
* **Result:** Total liabilities are compressed by 75%, collateralization restores to 200%, and leverage resets to **2.0×**:
  $$CR_{\text{new}} = \frac{C^{\text{rem}}}{V_A^{\text{rem}}} = \frac{\$0.50}{\$0.25} = 200\% \implies \lambda_{\text{new}} = \frac{\$0.50}{\$0.25} = \mathbf{2.0\times}$$

```
DOWNWARD RESET BALANCE SHEET WALKTHROUGH:
─────────────────────────────────────────────────────────────────────────────
1. Pre-Reset State (at H_d = $0.25 barrier):
   • Vault Collateral:  $1.25 in sAVAX
   • Senior Liability:  $1.00
   • Junior Equity:     $0.25

2. Debt Amortization:
   • Pay senior claim $0.75 in physical sAVAX.
   • Remaining Vault Collateral:  $1.25 - $0.75 = $0.50.
   • Remaining Senior Liability:  $1.00 - $0.75 = $0.25.
   • Remaining Junior Equity:    $0.25.

3. Share Consolidation (4-to-1 Reverse Split):
   • 4 old units consolidate into 1 new unit normalized to $1.00 par.
   • Post-Reset Vault Balance Sheet: $0.50 collateral backing $0.25 debt + $0.25 equity.
   • Collateralization Ratio: ($0.50 / $0.25) = 200%.
   • Effective Leverage: ($0.50 / $0.25) = 2.0x.
─────────────────────────────────────────────────────────────────────────────
```

---

## 3. Staking Yield Transmission & Capital Waterfall

### 3.1 Gross Consensus Yield & Surplus Equations

Because $2.00 of sAVAX collateral is pledged for every $1.00 of senior debt, the protocol captures consensus staking yield on **twice the notional debt base**:

$$\text{Gross Staking Yield} = 2.00 \cdot r_{\text{sAVAX}}$$

For an illustrative sAVAX consensus reward rate of $r_{\text{sAVAX}}$ between 4.5% and 5.5% p.a., gross cash flow across the collateral base equals **9.0% to 11.0% p.a.** relative to senior notional debt.

The protocol surplus cash flow per dollar of senior debt is defined as:

$$\Pi_{\text{surplus}} = 2.00 \cdot r_{\text{sAVAX}} - R$$

For $r_{\text{sAVAX}} = 5.0\%$ and $R = 7.3\%$:
$$\Pi_{\text{surplus}} = 2(5.0\%) - 7.3\% = \mathbf{2.7\% \text{ per dollar of senior debt}}$$

This cash flow feeds a transparent, structured yield transmission waterfall:

```
[ Gross sAVAX Staking Cash Flow: ~9.0% - 11.0% ]
                        │
       ┌────────────────┴────────────────┐
       ▼                                 ▼
[ 1. Senior Fixed Obligations ]   [ 2. Protocol Surplus Yield ]
       R = 7.3% p.a.                      ~2.0% - 4.0% p.a.
       │                                  │
       ├─ anUSD base rate: 2.5%           └─ Strategic Allocation Options:
       └─ anYIELD coupon: 12.1%              • Accrual to QI stakers
                                             • sAVAX/AVAX DEX pool depth
                                             • Internal risk reserve buffer
```

### 3.2 Protocol Surplus Governance & Strategic Flexibility

The October 2026 Executive Brief referenced a surplus allocation of 65% buyback and burn, 20%–45% validator security boost, and 15% ecosystem grants.

**We clarify that this distribution was modeled on an informal proposal from Avalanche Community Proposal (ACP-67) discussions as an illustrative framework for Avalanche Foundation alignment. It does not represent an architectural constraint or mandate for BENQI.**

From BENQI’s perspective, protocol surplus is fully modular and controlled via protocol governance. Potential allocation pathways include:
* **Accrual to QI Stakers:** Routing surplus cash flows directly to $QI lockers or stakers.
* **DEX Liquidity Incentives:** Subsidizing sAVAX/AVAX liquidity depth on Pharaoh or LFJ to tighten secondary peg efficiency.
* **On-Chain Risk Reserves:** Capitalizing an internal emergency liquidity pool to backstop secondary market redemption dynamics.

---

## 4. Downside Stress Analysis: Multi-Week Contractions vs. Flash Crashes

A central consideration in evaluating anUSD is system resilience during extreme collateral drawdowns—both sustained multi-week bear trends and single-block jump discontinuities.

### 4.1 Reconciling the 50% Primary Pool Threshold with 75% Par Crash Invariance

A standard review of the initial balance sheet ($2.00 collateral backing $1.00 senior debt) prompts an immediate structural question:
* *If AVAX drops 50%, collateral contracts from $2.00 to $1.00, exhausting the junior equity buffer. How does the architecture mathematically support Theorem 1's claim of 75% flash crash invariance?*

The resolution lies in the secondary tranche subordination:

1. **Primary Tranche Limit (Single Pair):**
   * Evaluating 1 unit of anBOND backed by $2.00 of sAVAX in isolation, a **50% drop** ($S_t = 0.50$) brings collateral to parity with debt:
     $$\text{Collateral Remaining} = 2.00 \times 0.50 = 1.00\text{ USD}$$
     Any decline exceeding 50% absent a reset impairs the primary debt tranche.
2. **Secondary Tranche Subordination (anUSD Invariance):**
   * Minting 1 unit of anUSD requires locking **2 units of anBOND**.
   * Consequently, **$4.00 of gross collateral supports every $1.00 of issued anUSD** ($2.00 in junior anAVAX equity, and $1.00 in subordinated anYIELD).
   * In an instantaneous **75% flash crash** ($S_t = 0.25$):
     $$\text{Collateral Remaining} = 4.00 \times 0.25 = 1.00\text{ USD}$$
   * Collateral remains exactly equal to $1.00 per unit of anUSD. Therefore, **anUSD sustains an instantaneous 75% market drop from par with zero haircut to principal**.

```
PAR CRASH SUBORDINATION WALKTHROUGH ($4.00 Gross Collateral backing $1.00 anUSD):
─────────────────────────────────────────────────────────────────────────────
• Baseline State:  $4.00 collateral  ──>  $1.00 anUSD + $1.00 anYIELD + $2.00 anAVAX
• Instant -50%:    $2.00 collateral  ──>  anAVAX fully absorbed. anUSD & anYIELD 100% intact.
• Instant -75%:    $1.00 collateral  ──>  anYIELD fully absorbed. anUSD remains 100% covered ($1.00 par).
─────────────────────────────────────────────────────────────────────────────
```

### 4.2 Multi-Week Bear Market Dynamics (30-Day 50%–60% Drawdown)

In a progressive multi-week contraction (e.g., AVAX declining 50%–60% over 30 days):
* The protocol does not wait for equity exhaustion. Upon reaching the $H_d = 0.25$ threshold (−37.5% drop), the downward reset executes deterministically.
* 75% of senior liabilities are amortized in physical sAVAX, the remaining capital base is consolidated via the 4:1 reverse split, and the vault enters the next market phase with fresh 200% collateral coverage and 2.0× leverage.
* Successive market drops trigger sequential resets, maintaining senior coverage across extended bear regimes.

### 4.3 Simulation Scope & Epistemic Boundaries

To maintain research rigor, we clearly distinguish between verified modeling and unsimulated operational domains:

* **Simulated & Validated Domains:**
  * 500 Kou jump-diffusion paths parameterized on 2,140 days of empirical Avalanche price volatility ($\sigma = 89.15\%$).
  * Resets executed cleanly under programmed boundaries, yielding **0.00% senior haircut probability** with an average reset frequency of 3.71 events per annum.
* **Unsimulated Frontiers Requiring Empirical Modeling:**
  * We have **not** modeled a multi-agent deterministic 30-day crash scenario incorporating behavioral feedback loops following consecutive resets.
  * We have **not** simulated potential secondary market price slippage if amortized senior holders immediately liquidate received sAVAX on DEXs.
  * We have **not** modeled P-Chain unstaking queue delays under correlated network-wide withdrawal runs.

---

## 5. Comparative Mechanism Architecture

| Mechanism | Loss Absorption Framework | Primary Systemic Failure Mode |
| :--- | :--- | :--- |
| **MakerDAO / Sky** | Asynchronous Dutch liquidation auctions | Mempool congestion and auction slippage (e.g., Black Thursday $4.5M bad debt from 0-bid auctions) |
| **Liquity (LUSD/BOLD)** | Stability Pool absorption & borrower interest rates | Stability Pool depletion during prolonged multi-month downturns |
| **Terra / UST** | Mint-and-burn arbitrage against endogenous LUNA | Reflexive hyper-inflationary death spiral upon confidence loss |
| **anUSD** | Capital subordination + deterministic balance-sheet resets | Secondary DEX discounts, oracle latency, and unstaking queue congestion |

*(Research Scope Note: Historical BCRG evaluations focused on MakerDAO, Liquity, and Terra. Delta-neutral synthetic designs such as Ethena USDe have not been formally modeled in our pipeline and are excluded from comparative claims).*

---

## 6. Open Engineering & Modeling Questions

While the baseline balance-sheet equations and jump-diffusion boundaries are verified, transitioning this mechanism into a production protocol raises several real-world engineering questions that require dedicated modeling:

### 6.1 Secondary DEX Discounts (The sAVAX De-Peg Risk)

Our baseline balance-sheet equations assume 1 sAVAX can always be redeemed or liquidated for its exact theoretical value. In reality:
* During sharp market crashes, users frequently sell sAVAX directly on secondary DEXs (Pharaoh, LFJ) rather than waiting for unstaking queues.
* If sAVAX trades at a 4%–6% secondary discount (e.g. trading at $0.94\text{ AVAX}$), effective vault collateral value contracts faster than spot AVAX, compressing the equity cushion.
* *Open Modeling Question:* Does the $0.25 junior equity buffer remain thick enough to protect senior debt if a 37.5% spot price drop coincides with a 6% secondary DEX discount?

### 6.2 Preventing "Ping-Pong" Resets (Secondary Selling Pressure Feedback)

In a downward reset, the protocol pays out $0.75 of physical sAVAX per token to senior holders.
* If those senior holders immediately turn around and sell their sAVAX on DEXs to obtain cash, that selling pressure could push AVAX market prices down further.
* If this secondary market impact is large, it could trigger a second downward reset shortly after the first.
* *Open Modeling Question:* How large can the physical payout fraction ($\gamma$) be without triggering secondary price pressure that causes repeated, ping-ponging resets?

### 6.3 14-Day P-Chain Unstaking Queue vs. Instant Reserves

BENQI maintains an instant unstaking pool on C-Chain to save users from the 14-day P-Chain unstaking queue.
* In a severe downturn, if senior holders receiving physical sAVAX all attempt to withdraw to native AVAX at once, BENQI’s instant unstaking pool could be exhausted.
* Holders would then enter the 14-day P-Chain unstaking queue.
* *Open Modeling Question:* How does the protocol maintain market confidence and anUSD peg stability during an extended 14-day unbonding delay?

---

### 6.4 The Governance Trade-Off Matrix

The mechanism is governed by seven interconnected parameters:

$$\theta = \left( H_u, \; H_d, \; R, \; R', \; \gamma, \; \vec{\omega}_{\text{surplus}}, \; \Delta t_{\text{oracle}} \right)$$

Selecting these values involves multi-dimensional trade-offs between capital efficiency, user friction, and tail-risk solvency:

| Parameter | Low Setting Trade-Off | High Setting Trade-Off | Dominant Systemic Risk |
| :--- | :--- | :--- | :--- |
| **Upper Boundary ($H_u$)** | Triggers frequent forward splits; clips leveraged upside for anAVAX; increases gas overhead and rebalancing friction. | Allows leverage to decay heavily (below 1.5×); suppresses junior equity distributions; reduces capital turnover. | Upside leverage decay vs. execution overhead. |
| **Lower Boundary ($H_d$)** | High leverage expansion (up to 5×+); increases jump-risk vulnerability if a market discontinuity strikes. | Frequent downward resets; premature debt amortization; user friction and contract rebalancing overhead. | Flash crash haircut risk (Theorem 1 boundary) vs. rebalancing fatigue. |
| **Senior Coupon ($R$)** | Uncompetitive yield for institutional anBOND capital; slow initial TVL scaling. | Compresses protocol surplus; risks negative net carry if base staking APR declines. | Staking spread solvency ($2 \cdot r_{\text{sAVAX}} > R$). |
| **anUSD Base Yield ($R'$)** | Lower initial adoption of anUSD as an ecosystem payment and settlement currency. | Depletes the residual coupon captured by anYIELD ($2R - R'$); disincentivizes high-yield capital. | Tranche balance: preserving an attractive spread for both anUSD and anYIELD. |
| **Debt Amortization ($\gamma$)** | Leaves high residual senior debt in vault; weaker balance-sheet recapitalization post-reset. | Sizable physical sAVAX distributed to users; risks immediate secondary DEX selling pressure. | Balance-sheet deleveraging vs. secondary market sAVAX discount contagion. |
| **Surplus Vector ($\vec{\omega}$)** | Directing yield primarily to insurance reserves depresses near-term QI token cash-flow accrual. | High distribution to QI stakers leaves the protocol under-buffered during black-swan liquidity shocks. | Token value accrual vs. long-term balance-sheet resilience. |

---

### 6.5 Parameter Optimization (PSUU)

The seven governing parameters are non-linearly coupled and cannot be calibrated through static spreadsheet models:
* Widening reset boundaries ($H_d < 0.25$) expands peak leverage, requiring either a higher debt amortization fraction ($\gamma > 0.75$) or a higher equity buffer to maintain flash-crash invariance.
* Calibrating $R$ without stochastic modeling of consensus reward fluctuations risks balance-sheet margin compression if network staking yield drops from 5.0% to 3.5%.

To determine safe production parameters, a formal **Parameter Selection Under Uncertainty (PSUU)** simulation pipeline is necessary:
1. **Admissible Parameter Space:** Define the valid mathematical parameter regions where Theorem 1 flash-crash invariance strictly holds.
2. **Stochastic Volatility Sweeps:** Simulate 10,000+ price paths across historical Avalanche volatility ($\sigma \approx 89\%$) and jump regimes.
3. **Liquidity Feedback Stress Testing:** Evaluate parameter combinations against correlated DEX discounts and P-Chain queue delays.
4. **Pareto Optimization:** Identify parameter sets that simultaneously minimize senior haircut probability ($\approx 0$), minimize reset churn ($\le 2$–$3$ events per year), and maximize sustainable economic yield for the BENQI ecosystem.

---

## 7. Status & Technical Availability

This memorandum summarizes the foundational mechanics and current research findings for the anUSD design in response to preliminary technical review questions. 

As BENQI continues its evaluation of the mechanism, BCRG remains available to address further technical questions or discuss the analytical methodology behind these preliminary models.

***

*Bonding Curves Research Group (BCRG)*
