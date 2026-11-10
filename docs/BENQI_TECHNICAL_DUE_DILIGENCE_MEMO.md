# anUSD Technical Due Diligence Memorandum
## Architectural Specifications, Yield Waterfall, and Stress Dynamics

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

## 2. Vault Architecture & Dynamic Leverage Mechanics

### 2.1 Collateral Decomposition: Senior Debt and Junior Equity

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

* **anBOND (Senior Tranche):** Represents a senior contractual claim on the underlying sAVAX collateral. It prioritizes principal preservation and receives an annual coupon rate $R$ (modeled initially at 7.3% p.a.).
* **anAVAX (Junior Tranche):** Represents the residual equity claim. It absorbs collateral fluctuations on a first-loss basis. Because $100 of junior equity supports $200 of gross collateral, **it initiates with exactly 2.0× leveraged exposure to AVAX price movements**.

---

### 2.2 Secondary Tranche Subordination: Mint and Redeem Rules for anUSD and anYIELD

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

* **anUSD:** The unit-pegged stable monetary instrument. It maintains a $1.00 target and may distribute an optional base money-market yield $R'$ (e.g., 2.5% p.a.).
* **anYIELD:** A subordinated yield-concentrating instrument. It captures the residual coupon of the underlying anBOND units:
  $$R_{\text{anYIELD}} = 2R - R'$$

At baseline reference rates ($R = 7.3\%$ and $R' = 2.5\%$ p.a.), the resulting yield on anYIELD evaluates to **12.1% p.a.** (calculated as 2 × 7.3% − 2.5% = 12.1%).

**Minting and Redemption Conversion Rules:**
* **Issuance (Mint):** Locking **2 units of anBOND** mints **1 unit of anUSD + 1 unit of anYIELD**.
* **Settlement (Redeem):** Surrendering **1 unit of anUSD + 1 unit of anYIELD** redeems **2 units of anBOND**.

*(Accounting Note: Final technical specifications must confirm whether the 2:1 ratio is enforced strictly in unit quantities or dollar-equivalent nominal value, ensuring strict value conservation across multi-epoch operations).*

---

### 2.3 Dynamic Leverage Evolution Across Price Trajectories

Leverage in the junior tranche is not static; it responds continuously to changes in underlying collateral value:

$$\text{Leverage} = \frac{\text{Total Collateral Value}}{\text{anAVAX Equity Value}}$$

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

* **Collateral Appreciation (+50%):** Collateral value expands to $3.00. The senior claim remains $1.00, increasing junior equity to $2.00. Effective leverage naturally compresses from **2.0× to 1.5×**.
* **Collateral Depreciation (−37.5%):** Collateral value contracts to $1.25. Subtracting the $1.00 senior claim leaves $0.25 in junior equity. Effective leverage expands from **2.0× to 5.0×**.

To bound leverage expansion and protect senior solvency without resorting to external liquidation auctions, the protocol executes automated, deterministic balance-sheet resets at predefined boundaries.

---

### 2.4 Internal Reset & Rebalancing Mechanics (The Share Splits)

The October 2026 Executive Brief noted that anUSD provides contagion protection because *"shares consolidate internally"* during severe drawdowns. Below are the formal mechanics governing both the downward consolidation (reverse split) and upward harvest (forward split):

#### 1. Upward Reset (Triggered at $H_u = 2.00$ / +50% AVAX Gain)
* **Objective:** Realize accumulated equity gains and restore the junior tranche to target 2.0× leverage.
* **Mechanism:** 
  1. The protocol crystallizes junior capital gains and distributes accumulated profit in physical sAVAX directly to anAVAX holders.
  2. The contract executes a **2-for-1 forward share split**, restoring the per-unit price of anAVAX back to $1.00.
* **Result:** System leverage cleanly recalibrates to **2.0×** without altering senior debt obligations.

#### 2. Downward Reset (Triggered at $H_d = 0.25$ / −37.5% AVAX Drop)
* **Objective:** Eliminate junior leverage blowout, amortize senior liabilities, and restore balance-sheet collateralization prior to any senior capital impairment.
* **Mechanism:**
  1. **Senior Debt Amortization:** The contract automatically settles **$0.75 of each $1.00 senior claim directly in physical sAVAX**, extinguishing 75% of total protocol liabilities.
  2. **Capital Consolidation (4-for-1 Reverse Split):** The remaining $0.25 senior claim and $0.25 junior equity claim undergo an internal 4-for-1 reverse split, merging 4 old shares into 1 new share with a normalized par value of $1.00.
* **Result:** Total liabilities are dramatically compressed, the vault re-establishes a 200% collateral ratio ($0.50 collateral backing $0.25 debt), and effective leverage resets to **2.0×**.

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

### 3.1 Gross Consensus Yield Amplification

Because $2.00 of sAVAX collateral is pledged for every $1.00 of senior debt, the protocol captures consensus staking yield on **twice the notional debt base**:

$$\text{Gross Staking Yield} = 2 \times r_{\text{sAVAX}}$$

For an illustrative sAVAX consensus reward rate of $r_{\text{sAVAX}}$ between 4.5% and 5.5% p.a., gross cash flow across the collateral base equals **9.0% to 11.0% p.a.** relative to senior notional debt.

This gross cash flow feeds a transparent, structured yield transmission waterfall:

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

## 6. Current Research Boundaries & Open Modeling Areas

While the baseline mathematical architecture and jump-diffusion boundaries are established, transitioning this mechanism into a production protocol requires addressing several critical unmodeled dynamics:

### 6.1 The sAVAX vs. AVAX Market Relationship

Our baseline proofs assume 1 sAVAX redeems at its exact theoretical exchange rate without market friction. In reality, the dynamic coupling between sAVAX and AVAX introduces three distinct risk channels that have not yet been formally modeled:

1. **Secondary DEX Discounts:** During sharp market panics, users frequently sell sAVAX on DEXs (Pharaoh, LFJ) rather than waiting for unstaking queues. If sAVAX trades at a discount (e.g., $0.94\text{ AVAX}$), effective vault collateral value contracts faster than spot AVAX, compressing the equity cushion.
2. **Unstaking Queue Latency:** In a downward reset, senior holders receive 75% of their principal in physical sAVAX. If a significant cohort attempts to unbond to native AVAX simultaneously, immediate C-Chain liquidity reserves could be depleted, forcing holders into the 14-day P-Chain unstaking queue. The market impact of this latency requires dedicated modeling.
3. **Staking APR Volatility:** Network-wide staking yield fluctuates with total staked AVAX and network activity. If the underlying sAVAX staking yield compresses from 5% to 3%, senior coupon rates ($R, R'$) must adapt dynamically to avoid margin compression.

### 6.2 Parameter Selection Under Uncertainty (PSUU)

The numerical values referenced in this note:
* Reset boundaries: $H_u = 2.00$ (+50% gain), $H_d = 0.25$ (−37.5% drop)
* Senior coupon: $R = 7.3\%$ p.a.
* anUSD base yield: $R' = 2.5\%$ p.a.

These represent **preliminary reference starting points** used for structural illustration, not optimized deployment values.

A rigorous multi-variable parameter optimization (PSUU) has not yet been executed. Developing an empirical parameter surface requires modeling several structural trade-offs:
* **Reset Frequency vs. Execution Cost:** Narrower boundaries reduce leverage drift but trigger resets more frequently (~3.7/year), increasing on-chain execution overhead. Wider boundaries reduce reset frequency (~1–2/year) but allow leverage to expand higher, widening jump risk exposure.
* **Yield Spread Calibration:** Calibrating the sustainable spread between gross staking cash flow and senior coupons under stochastic staking yield regimes.
* **Collateral Buffer Sizing:** Sizing the junior equity cushion against correlated market crashes and secondary DEX peg deviations.

---

## 7. Status & Technical Availability

This memorandum summarizes the foundational mechanics and current research findings for the anUSD design in response to preliminary technical review questions. 

As BENQI continues its evaluation of the mechanism, BCRG remains available to address further technical questions or discuss the analytical methodology behind these preliminary models.

***

*Bonding Curves Research Group (BCRG)*
