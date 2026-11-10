# anUSD Technical Questions & Answers
## Mechanism Mechanics, Balance Sheet Accounting & Downside Dynamics

**To:** Hansen Niu & BENQI Strategy Team  
**From:** Bonding Curves Research Group (BCRG)  
**Date:** November 10, 2026  
**Reference:** *anUSD Executive Proposal (shared November 9, 2026)*  
**Topic:** Follow-up Technical Q&A

---

## 1. Context & Questions

Following the *anUSD Executive Proposal* shared yesterday, Hansen raised four practical technical questions on Telegram regarding how the mechanism functions under the hood:

1. **Tranche Mechanics & Dynamic Leverage:** How mint/redeem rules work, collateral ratios, and how 2.0x leverage is maintained without liquidation auctions.
2. **Staking Yield Split:** How sAVAX staking rewards are divided between the fixed senior coupon (anBOND), high-yield tranche (anYIELD), and protocol surplus.
3. **Downside Crash Behavior:** How the vault behaves during a prolonged bear market (e.g., AVAX falling 50%–60% over a month) versus an instantaneous flash crash.
4. **Comparative Architecture:** How structural risks compare against MakerDAO/Sky, Liquity, and Terra.

This note answers each question directly using straightforward balance-sheet accounting, walks through the numbers step-by-step, and outlines the open modeling questions where further research is needed.

---

## 2. Vault Architecture & Balance Sheet Equations

### 2.1 The Core Balance Sheet: Senior Debt vs. Junior Equity

Rather than borrowing against collateral in a CDP and risking auction liquidation, anUSD pools deposited sAVAX and divides the pool into two complementary claims:

```
[ Vault Deposits: $200 of sAVAX ]
               │
   ┌───────────┴───────────┐
   ▼                       ▼
[ anBOND: Senior Debt ]  [ anAVAX: Junior Equity ]
        $100                    $100
 • Fixed claim: $1.00     • Absorbs all price swings
 • Earns 7.3% coupon      • Starts with 2.0x leverage
                          ($100 equity controls $200 collateral)
```

The vault accounting follows four foundational rules:

---

#### Rule 1: Total Collateral Value ($C_t$)
The vault holds twice as much collateral as the initial senior debt:

$$\Large \displaystyle C_t = 2.00 \times S_t$$

*What this means:* Let $S_t$ be the AVAX price index (normalized to 1.00 USD at the start of an epoch). If AVAX starts at 1.00 USD, the vault holds 2.00 USD of sAVAX collateral. If AVAX rises 50% to 1.50 USD, the collateral is worth 3.00 USD. If AVAX drops 37.5% to 0.625 USD, collateral drops to 1.25 USD.

---

#### Rule 2: Senior Debt Claim ($V_A$)
The senior tranche (anBOND) has a priority claim that starts at par and earns a contractual coupon $R$ (modeled at 7.3% p.a.):

$$\Large \displaystyle V_A = 1.00 \text{ USD}$$

*What this means:* Regardless of whether AVAX goes up or down, senior holders are owed their fixed 1.00 USD principal plus accrued interest.

---

#### Rule 3: Junior Equity Claim ($V_B$)
The junior tranche (anAVAX) absorbs whatever collateral remains after paying the senior claim:

$$\Large \displaystyle V_B = C_t - V_A = 2.00 \times S_t - 1.00$$

*What this means:* Because the senior claim stays fixed at 1.00 USD, any gain or loss on the underlying collateral flows 100% into junior equity:
* If collateral rises to 3.00 USD, junior equity doubles to: $3.00 - 1.00 = \mathbf{2.00 \text{ USD}}$.
* If collateral falls to 1.25 USD, junior equity shrinks to: $1.25 - 1.00 = \mathbf{0.25 \text{ USD}}$.

---

#### Rule 4: Dynamic Junior Leverage ($\lambda$)
Effective leverage is the ratio of gross collateral to junior equity:

$$\Large \displaystyle \lambda = \frac{C_t}{V_B} = \frac{2.00 \times S_t}{2.00 \times S_t - 1.00}$$

*What this means:*
* **At inception:** 2.00 USD collateral / 1.00 USD equity = **2.0× leverage**.
* **When AVAX rises 50%:** 3.00 USD collateral / 2.00 USD equity = **1.5× leverage** (leverage naturally decays as equity grows).
* **When AVAX falls 37.5%:** 1.25 USD collateral / 0.25 USD equity = **5.0× leverage** (leverage expands as equity shrinks).

---

### 2.2 Secondary Tranche Subordination: anUSD and anYIELD

Because anBOND accrues coupon interest over time, its value naturally drifts above $1.00 (to $1.01, $1.02, etc.). To create a stable, unit-pegged currency for payments and treasury reserves, anBOND is partitioned into two tokens:

```
              [ 2 Units of anBOND ($2.00 total par) ]
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
       [ anUSD Stablecoin ]           [ anYIELD High-Yield ]
             $1.00 par                      $1.00 par
       • Pegged at $1.00              • Collects concentrated coupon
       • Optional base yield R'       • Formula: R_anYIELD = 2R - R'
         (e.g., 2.5% p.a.)              (e.g., 12.1% p.a.)
```

The secondary tranche conversion follows two conservation rules:

---

#### Rule 1: Par Invariance (Value Conservation)

$$\Large \displaystyle P_{\text{anUSD}} + P_{\text{anYIELD}} = 2 \times P_{\text{anBOND}} = 2.00 \text{ USD}$$

*What this means:* The two tokens together always equal the value of the 2 locked anBOND units (1.00 USD + 1.00 USD = 2.00 USD).

---

#### Rule 2: Yield Concentration

$$\Large \displaystyle R_{\text{anYIELD}} = 2R - R'$$

*What this means:* The 2 units of anBOND generate $2 \times 7.3\% = 14.6\%$ in total annual coupon income. If anUSD pays out a 2.5% base money-market yield to holders, the remaining yield flows entirely to anYIELD:

$$\Large \displaystyle 14.6\% - 2.5\% = \mathbf{12.1\% \text{ p.a.}}$$

**Minting and Redemption Conversion Rules:**
* **To Mint:** Lock **2 units of anBOND** to mint **1 unit of anUSD + 1 unit of anYIELD**.
* **To Redeem:** Burn **1 unit of anUSD + 1 unit of anYIELD** together to unlock **2 units of anBOND**.

---

### 2.3 Dynamic Leverage Evolution Across Price Trajectories

Leverage in the junior tranche is not static; it moves continuously with the market:

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

* **When AVAX Rises +50% ($S_t = 1.50$):** Collateral value expands to 3.00 USD. Senior debt remains 1.00 USD, so junior equity doubles to 2.00 USD. Effective leverage naturally compresses:

$$\Large \displaystyle \lambda = \frac{3.00}{2.00} = \mathbf{1.5\times}$$

* **When AVAX Drops −37.5% ($S_t = 0.625$):** Collateral contracts to 1.25 USD. Subtracting the 1.00 USD senior claim leaves 0.25 USD in junior equity. Effective leverage expands:

$$\Large \displaystyle \lambda = \frac{1.25}{0.25} = \mathbf{5.0\times}$$

To prevent runaway leverage from creating solvency risk, the contract executes automated resets whenever junior equity reaches predefined boundaries.

---

### 2.4 Internal Reset & Rebalancing Mechanics (The Share Splits)

The Executive Proposal noted that anUSD eliminates auction contagion because *"shares consolidate internally"* during severe drawdowns. Below are the formal balance-sheet mechanics:

#### 1. Upward Reset (Triggered when AVAX rises +50%, so anAVAX hits $2.00)
* **What happens:** Junior equity holders have doubled their money ($V_B = 2.00 \text{ USD}$).
* **Step 1 (Profit Distribution):** The contract crystallizes the 1.00 USD profit per share and distributes it directly to junior holders in physical sAVAX.
* **Step 2 (Forward Split 2-for-1):** The contract executes a 2-for-1 forward share split, doubling total shares and restoring each share's price back to 1.00 USD.
* **Result (Reset Leverage):** With 2.00 USD of collateral and 1.00 USD of equity, leverage resets cleanly back to **2.0×**.

---

#### 2. Downward Reset (Triggered when AVAX drops −37.5%, so anAVAX hits $0.25)
* **What happens:** Junior equity has shrunk to 25 cents. The contract steps in to protect senior debt *before* market losses can touch it.
* **Step 1 (Senior Debt Amortization):** The contract automatically pays senior holders 75 cents of their 1.00 USD claim directly in physical sAVAX, extinguishing 75% of protocol liabilities:

$$\Large \displaystyle \text{Remaining Senior Debt} = 1.00 - 0.75 = 0.25 \text{ USD}$$

$$\Large \displaystyle \text{Remaining Vault Collateral} = 1.25 - 0.75 = 0.50 \text{ USD}$$

* **Step 2 (Capital Consolidation: 4-for-1 Reverse Split):** The remaining 25 cents of debt and 25 cents of equity merge via a 4-to-1 reverse split (4 old shares merge into 1 new share worth 1.00 USD).
* **Result (New Leverage & Collateral Ratio):**

$$\Large \displaystyle \text{Collateral Ratio} = \frac{0.50 \text{ Collateral}}{0.25 \text{ Debt}} = \mathbf{200\%}$$

$$\Large \displaystyle \text{Leverage} = \frac{0.50 \text{ Collateral}}{0.25 \text{ Equity}} = \mathbf{2.0\times}$$

*What this means:* The protocol has eliminated 75% of its debt, restored a 200% collateral cushion, and reset leverage back to 2.0×—all through internal accounting, without selling a single token in a liquidation auction.

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

Because the vault holds $2.00 of sAVAX collateral for every $1.00 of senior debt, the protocol captures consensus staking yield on **twice the notional debt base**:

$$\Large \displaystyle \text{Gross Staking Yield} = 2.00 \times r_{\text{sAVAX}}$$

*What this means:* If sAVAX consensus staking earns 5.0% p.a., the vault generates:

$$\Large \displaystyle 2 \times 5.0\% = \mathbf{10.0\% \text{ gross annual yield}}$$

The protocol surplus is whatever staking yield remains after paying the senior coupon $R$:

$$\Large \displaystyle \text{Protocol Surplus} = \text{Gross Staking Yield} - R$$

Using our baseline coupon of $R = 7.3\%$ p.a.:

$$\Large \displaystyle \text{Surplus} = 10.0\% - 7.3\% = \mathbf{2.7\% \text{ per dollar of senior debt}}$$

*What this means:* On a $100M anUSD vault, the protocol generates **$2.7M annually in surplus cash flow** that BENQI governance can allocate.

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

The Executive Proposal referenced an illustrative surplus allocation of 65% buyback and burn, 20%–45% validator security boost, and 15% ecosystem grants.

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
* *If AVAX drops 50%, collateral contracts from $2.00 to $1.00, exhausting the junior equity buffer. How does the architecture support the claim of 75% flash crash invariance from the Executive Proposal?*

The resolution lies in the secondary tranche subordination:

---

#### 1. Primary Tranche Limit (Single Pair)
Evaluating 1 unit of anBOND backed by 2.00 USD of sAVAX in isolation, a **50% drop** brings collateral down to parity with debt:

$$\Large \displaystyle \text{Collateral Remaining} = 2.00 \times 0.50 = 1.00 \text{ USD}$$

Any decline exceeding 50% absent a reset impairs the primary debt tranche.

---

#### 2. Secondary Tranche Subordination (anUSD Invariance)
Minting 1 unit of anUSD requires locking **2 units of anBOND**.
Consequently, **4.00 USD of gross collateral supports every 1.00 USD of issued anUSD** (2.00 USD in junior anAVAX equity, and 1.00 USD in subordinated anYIELD).

In an instantaneous **75% flash crash** (AVAX index drops to 0.25 USD):

$$\Large \displaystyle \text{Collateral Remaining} = 4.00 \times 0.25 = 1.00 \text{ USD}$$

Collateral remains exactly equal to 1.00 USD per unit of anUSD. Therefore, **anUSD sustains an instantaneous 75% market drop from par with zero haircut to principal**.

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
* The protocol does not wait for equity exhaustion. Upon reaching the $0.25 barrier (−37.5% drop), the downward reset executes deterministically.
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
* If sAVAX trades at a 4%–6% secondary discount (e.g. trading at 0.94 AVAX), effective vault collateral value contracts faster than spot AVAX, compressing the equity cushion.
* *Open Modeling Question:* Does the $0.25 junior equity buffer remain thick enough to protect senior debt if a 37.5% spot price drop coincides with a 6% secondary DEX discount?

### 6.2 Preventing "Ping-Pong" Resets (Secondary Selling Pressure Feedback)

In a downward reset, the protocol pays out 75 cents of physical sAVAX per token to senior holders.
* If those senior holders immediately turn around and sell their sAVAX on DEXs to obtain cash, that selling pressure could push AVAX market prices down further.
* If this secondary market impact is large, it could trigger a second downward reset shortly after the first.
* *Open Modeling Question:* How large can the physical payout fraction be without triggering secondary price pressure that causes repeated, ping-ponging resets?

### 6.3 14-Day P-Chain Unstaking Queue vs. Instant Reserves

BENQI maintains an instant unstaking pool on C-Chain to save users from the 14-day P-Chain unstaking queue.
* In a severe downturn, if senior holders receiving physical sAVAX all attempt to withdraw to native AVAX at once, BENQI’s instant unstaking pool could be exhausted.
* Holders would then enter the 14-day P-Chain unstaking queue.
* *Open Modeling Question:* How does the protocol maintain market confidence and anUSD peg stability during an extended 14-day unbonding delay?

---

### 6.4 The Governance Trade-Off Matrix

The mechanism is governed by seven interconnected parameters:

1. **Upper Reset Barrier ($H_u = 2.00$):** Price barrier triggering equity payout and forward split (+50% AVAX gain).
2. **Lower Reset Barrier ($H_d = 0.25$):** Price barrier triggering debt amortization and reverse split (−37.5% AVAX drop).
3. **Senior Coupon ($R = 7.3\%$ p.a.):** Annual fixed interest accrued by anBOND.
4. **Stablecoin Base Yield ($R' = 2.5\%$ p.a.):** Optional yield distributed to anUSD holders.
5. **Senior Debt Payout Fraction ($\gamma = 0.75$):** Percentage of debt paid out in physical sAVAX at $H_d$.
6. **Surplus Allocation Vector ($\vec{\omega}$):** Split between $QI stakers, DEX liquidity, and protocol risk reserves.
7. **Oracle Evaluation Cadence ($\Delta t$):** Maximum staleness and update window for automated resets.

Selecting these values involves multi-dimensional trade-offs between capital efficiency, user friction, and tail-risk solvency:

| Parameter | Low Setting Trade-Off | High Setting Trade-Off | Dominant Systemic Risk |
| :--- | :--- | :--- | :--- |
| **Upper Boundary ($H_u$)** | Triggers frequent forward splits; clips leveraged upside for anAVAX; increases gas overhead and rebalancing friction. | Allows leverage to decay heavily (below 1.5×); suppresses junior equity distributions; reduces capital turnover. | Upside leverage decay vs. execution overhead. |
| **Lower Boundary ($H_d$)** | High leverage expansion (up to 5×+); increases jump-risk vulnerability if a market discontinuity strikes. | Frequent downward resets; premature debt amortization; user friction and contract rebalancing overhead. | Flash crash haircut risk (75% crash limit) vs. rebalancing fatigue. |
| **Senior Coupon ($R$)** | Uncompetitive yield for institutional anBOND capital; slow initial TVL scaling. | Compresses protocol surplus; risks negative net carry if base staking APR declines. | Staking spread solvency ($2 \times r_{\text{sAVAX}} > R$). |
| **anUSD Base Yield ($R'$)** | Lower initial adoption of anUSD as an ecosystem payment and settlement currency. | Depletes the residual coupon captured by anYIELD ($2R - R'$); disincentivizes high-yield capital. | Tranche balance: preserving an attractive spread for both anUSD and anYIELD. |
| **Debt Amortization ($\gamma$)** | Leaves high residual senior debt in vault; weaker balance-sheet recapitalization post-reset. | Sizable physical sAVAX distributed to users; risks immediate secondary DEX selling pressure. | Balance-sheet deleveraging vs. secondary market sAVAX discount contagion. |
| **Surplus Vector ($\vec{\omega}$)** | Directing yield primarily to insurance reserves depresses near-term QI token cash-flow accrual. | High distribution to QI stakers leaves the protocol under-buffered during black-swan liquidity shocks. | Token value accrual vs. long-term balance-sheet resilience. |

---

### 6.5 Parameter Optimization (PSUU)

The seven governing parameters are non-linearly coupled and cannot be calibrated through static spreadsheet models:
* Widening reset boundaries ($H_d < 0.25$) expands peak leverage, requiring either a higher debt amortization fraction or a higher initial equity buffer to maintain flash-crash invariance.
* Calibrating $R$ without stochastic modeling of consensus reward fluctuations risks balance-sheet margin compression if network staking yield drops from 5.0% to 3.5%.

To determine safe production parameters, a formal **Parameter Selection Under Uncertainty (PSUU)** simulation pipeline is necessary:
1. **Admissible Parameter Space:** Define the valid parameter regions where the 75% flash-crash invariance strictly holds.
2. **Stochastic Volatility Sweeps:** Simulate 10,000+ price paths across historical Avalanche volatility ($\sigma \approx 89\%$) and jump regimes.
3. **Liquidity Feedback Stress Testing:** Evaluate parameter combinations against correlated DEX discounts and P-Chain queue delays.
4. **Pareto Optimization:** Identify parameter sets that simultaneously minimize senior haircut probability ($\approx 0$), minimize reset churn ($\le 2$–$3$ events per year), and maximize sustainable economic yield for the BENQI ecosystem.

---

## 7. Current Status & Next Steps

This note provides the technical breakdown of the anUSD mechanism in response to Hansen's questions, along with our preliminary findings and the boundaries of what has been modeled so far.

As BENQI continues reviewing the design, we are happy to address any follow-up questions, walk through specific numbers, or discuss how we can collaborate on the parameter modeling and stress-testing.

***

*Bonding Curves Research Group (BCRG)*
