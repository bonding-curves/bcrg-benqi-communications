# anUSD: Mechanism, Yield Allocation, and Risk Considerations

**To:** Hansen Niu & the BENQI Team  
**From:** Bonding Curves Research Group (BCRG)  
**Date:** November 10, 2026  
**Status:** Working Draft for Technical & Strategic Discussion

---

## 1. Why We Are Writing This

Hansen raised four practical questions about how anUSD works under the hood:

1. How the two-tranche vault works and how 2× leverage is maintained.
2. How sAVAX staking rewards flow through the different tokens.
3. What happens if AVAX falls 50% to 60% in a month.
4. How this design compares to existing stablecoins like MakerDAO, Liquity, and Terra.

This note answers those four questions directly. We also walk through the balance sheet math, point out where our initial numbers are still provisional, and highlight what we still need to model together using real BENQI data.

---

## 2. How the Mechanism Works

### 2.1 The Core Idea: Splitting Collateral into Senior Debt and Junior Equity

Instead of borrowing against collateral and risking a liquidation auction, anUSD splits deposited sAVAX into two separate claims against the same collateral pool.

Here is a concrete example:

```
[ User Deposits $200 of sAVAX into the Vault ]
                     │
       ┌─────────────┴─────────────┐
       ▼                           ▼
[ Senior Claim: anBOND ]    [ Junior Equity: anAVAX ]
         $100                        $100
  • Aims for capital          • Takes first loss on drops
    preservation              • Captures leveraged upside
  • Earns fixed coupon        • 2.0x leverage at start
    (e.g., 7.3% p.a.)           ($100 equity controls $200 collateral)
```

* **anBOND (Senior Tranche):** The conservative claim. It gets paid first and earns a fixed annual coupon $R$ ($7.3\%$ in our starting model).
* **anAVAX (Junior Tranche):** The risk-taking equity claim. It absorbs price volatility. Because $\$100$ of anAVAX controls $\$200$ of collateral, **it starts with exactly 2.0× leverage on AVAX price moves**.

---

### 2.2 Creating the Stablecoin: anUSD and anYIELD

Because anBOND earns interest over time, its price naturally drifts above \$1.00 ($1.01, 1.02...$). To turn this into a stable, dollar-pegged currency, anBOND is split a second time:

```
                 [ 2 Units of anBOND ($2.00 par) ]
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
       [ anUSD Stablecoin ]           [ anYIELD High-Yield ]
             $1.00 par                      $1.00 par
       • Stable $1.00 peg             • Concentrates remaining yield
       • Optional base rate R'        • Earns: 2R - R' = 12.1% APY
         (e.g., 2.5% p.a.)
```

* **anUSD:** The actual stablecoin. It stays pegged to \$1.00 and can distribute a base money-market yield $R'$ (e.g., $2.5\%$).
* **anYIELD:** A yield-seeking token. It absorbs the remaining senior coupon:
  $$\text{Yield on anYIELD} = 2R - R' = 2(7.3\%) - 2.5\% = \mathbf{12.1\%}$$

**The Mint and Redeem Rule:**
* **To Mint:** You lock **2 units of anBOND** to get **1 anUSD + 1 anYIELD**.
* **To Redeem:** You burn **1 anUSD + 1 anYIELD** together to get back **2 units of anBOND**.

*(Note on Accounting: We need to confirm together whether this 2:1 ratio is tracked in raw token units or dollar-denominated value. This distinction matters for proving value conservation over long epochs).*

---

### 2.3 How 2× Leverage Actually Changes When Price Moves

Leverage is not fixed at 2× forever. It moves naturally with the market:

$$\text{Leverage} = \frac{\text{Total Collateral Value}}{\text{anAVAX Equity Value}}$$

Let's look at what happens starting from \$2.00 of collateral backing \$1.00 of senior debt and \$1.00 of junior equity:

```
   AVAX Drops -37.5%                      Starting State                       AVAX Rises +50%
─────────────────────────────────      ───────────────────────      ─────────────────────────────────
• Collateral falls to: $1.25           • Collateral: $2.00          • Collateral rises to: $3.00
• Senior claim takes:  $1.00           • Senior debt: $1.00         • Senior claim takes:  $1.00
• Junior equity left:  $0.25           • Junior equity: $1.00       • Junior equity left:  $2.00
─────────────────────────────────      ───────────────────────      ─────────────────────────────────
▶ LEVERAGE RISES TO: 5.0x              ▶ LEVERAGE IS: 2.0x          ▶ LEVERAGE DECAYS TO: 1.5x
  ($1.25 / $0.25)                        ($2.00 / $1.00)              ($3.00 / $2.00)
  [Triggers Downward Reset]                                           [Triggers Upward Reset]
```

* **When AVAX rises 50%:** Collateral grows to \$3.00. Senior debt is still \$1.00, so anAVAX doubles to \$2.00. Leverage naturally drops to **1.5×**.
* **When AVAX falls 37.5%:** Collateral drops to \$1.25. After the \$1.00 senior claim, anAVAX shrinks to \$0.25. Leverage jumps to **5.0×**.

If leverage keeps growing, a small price drop could wipe out the equity. To prevent runaway leverage without liquidation auctions, the protocol runs automatic resets.

---

### 2.4 The Reset Mechanism (No Liquidation Auctions)

Existing protocols like MakerDAO rely on liquidator bots bidding in auctions during crashes. When the network gets congested, those auctions fail. anUSD replaces auctions with **global contract resets** triggered at two fixed price barriers:

#### 1. Upward Reset (Triggered when anAVAX hits \$2.00 / +50% AVAX gain)
* **What happens:** Junior equity holders have doubled their money. The contract harvests this gain and pays it out directly in sAVAX to their wallets.
* **The reset action:** The protocol executes a 2-for-1 forward share split. Each token's unit price returns to \$1.00.
* **Outcome:** Leverage is cleanly reset back to **2.0×**.

#### 2. Downward Reset (Triggered when anAVAX hits \$0.25 / −37.5% AVAX drop)
* **What happens:** The contract protects senior debt before losses can touch it.
* **The reset action:**
  1. The contract pays senior holders **\$0.75 of their \$1.00 claim directly in sAVAX**. This immediately cuts total protocol debt by 75%.
  2. The remaining 25 cents of debt and 25 cents of equity are merged via a **4-for-1 reverse split**.
* **Outcome:** Every remaining token is restored to a clean \$1.00 par value, backed by 400% collateral coverage, and leverage resets back to **2.0×**.

```
DOWNWARD RESET BALANCE SHEET WALKTHROUGH:
─────────────────────────────────────────────────────────────────────────────
1. Pre-Reset State (at $0.25 barrier):
   • Vault Collateral:  $1.25 in sAVAX
   • Senior Liability:  $1.00
   • Junior Equity:     $0.25

2. Execution:
   • Pay senior holders $0.75 in physical sAVAX.
   • Vault collateral remaining: $1.25 - $0.75 = $0.50.
   • Senior liability remaining: $1.00 - $0.75 = $0.25.
   • Junior equity remaining:   $0.25.

3. Consolidation (4-to-1 Reverse Split):
   • 4 old shares merge into 1 new share worth $1.00.
   • New Vault State: $0.50 collateral backing $0.25 debt + $0.25 equity.
   • Collateral Ratio: ($0.50 / $0.25) = 200% (backed 2:1 again).
   • Leverage: ($0.50 / $0.25) = exactly 2.0x.
─────────────────────────────────────────────────────────────────────────────
```

---

## 3. How Staking Yield Is Divided

Because the vault holds \$2.00 of sAVAX for every \$1.00 of senior debt, the protocol earns **double the staking rewards** per dollar of debt:

$$\text{Gross Staking Yield} = 2 \times (4.5\% \text{ to } 5.5\%) = \mathbf{9.0\% \text{ to } 11.0\%}$$

This staking income flows down a structured waterfall:

```
[ Total sAVAX Staking Yield: ~9.0% - 11.0% ]
                     │
       ┌─────────────┴─────────────┐
       ▼                           ▼
[ 1. Senior Fixed Coupon ]  [ 2. Protocol Surplus Yield ]
       R = 7.3% p.a.                    ~2% - 4% p.a.
       │                                │
       ├─ anUSD base: 2.5%              └─ Strategic Decision for BENQI:
       └─ anYIELD:   12.1%                 • Distribute to QI stakers
                                           • Subsidize sAVAX/AVAX DEX pools
                                           • Build protocol insurance fund
```

### Clarification on the "ACP-67" 65/20/15 Split

Earlier materials referenced a surplus split of 65% buyback, 20–45% validator boost, and 15% grants. 

**We want to be completely clear: that was an informal suggestion taken from a GitHub comment on Avalanche Community Proposal ACP-67.** It was an example of what the Foundation might like, but **it is not a requirement or a fixed rule for BENQI.**

From BENQI’s perspective, any surplus revenue can be allocated wherever it creates the most commercial value for your protocol—for example:
* Directing yield to **QI token stakers**.
* Deepening **sAVAX liquidity pools** on Pharaoh or LFJ.
* Creating an on-chain **reserve buffer** for BENQI.

---

## 4. What Happens if AVAX Drops 50% to 60% in a Month?

Hansen specifically asked about a 50%–60% drawdown. To answer this clearly, we have to look at what the math says, what our simulation showed, and where we still need to be careful.

### 4.1 A Common Confusion: 50% vs. 75% Crash Protection

If an engineer looks at our basic starting table:
* \$2.00 collateral backing \$1.00 senior debt.

Their immediate reaction is: *"If AVAX drops 50%, collateral falls from \$2.00 to \$1.00. That leaves zero cushion. How can your paper claim 75% flash crash protection?"*

Here is the exact explanation:

1. **In the Simple Primary Pool (Single Pair):**
   * If you look *only* at 1 unit of anBOND backed by \$2.00 of sAVAX, a **50% drop** ($S_t = 0.50$) brings collateral down to exactly \$1.00 ($2 \times 0.50 = \$1.00$). Any drop beyond 50% would cause a shortfall if no reset happens.
2. **Why anUSD Survives a 75% Drop:**
   * To mint 1 unit of anUSD, you must burn **2 units of anBOND**.
   * That means **\$4.00 of deposited collateral sits behind every \$1.00 of anUSD** (\$2.00 in junior anAVAX equity, and \$1.00 in subordinated anYIELD).
   * At a **75% flash crash** ($S_t = 0.25$):
     $$\$4.00 \times 0.25 = \mathbf{\$1.00}$$
   * The remaining collateral still equals \$1.00, meaning **anUSD avoids any haircut even during an immediate 75% crash from par**.

```
PAR CRASH WALKTHROUGH ($4.00 Collateral backing $1.00 anUSD):
─────────────────────────────────────────────────────────────────────────────
• Initial state:   $4.00 collateral  ──>  $1.00 anUSD + $1.00 anYIELD + $2.00 anAVAX
• Instant -50%:    $2.00 collateral  ──>  anAVAX wiped out. anUSD & anYIELD still 100% covered.
• Instant -75%:    $1.00 collateral  ──>  anYIELD wiped out. anUSD is STILL 100% covered ($1.00 par).
─────────────────────────────────────────────────────────────────────────────
```

### 4.2 Gradual 30-Day Bear Market vs. Sudden Flash Crash

* **In a gradual multi-week decline:** The protocol doesn't wait for a 50% drop. As soon as AVAX falls −37.5%, the downward reset triggers automatically at the \$0.25 barrier. Senior debt is paid down by 75%, shares merge, and the vault recapitalizes with a fresh 2:1 cushion.
* **In an instant flash crash:** Theorem 1 proves that anUSD tolerates an instantaneous single-block drop of up to **−75% from par** and **−60% from the reset barrier** without losing a cent of principal.

### 4.3 What We Have Actually Modeled vs. What We Have Not

To be completely honest about the state of our research:

* **What we have modeled:**
  * We ran 500 Kou jump-diffusion paths over 365 days using 2,140 days of historical Avalanche price volatility ($\sigma = 89.15\%$).
  * In those simulations, resets executed as programmed, and **senior haircut probability was 0.00%** with an average of 3.71 resets per year.
* **What we have NOT modeled:**
  * We have **not** simulated a deterministic 30-day crash scenario with real market participants reacting to consecutive resets.
  * We have **not** modeled whether users receiving physical sAVAX during a reset immediately dump it on DEXs, creating secondary price pressure.
  * We have **not** modeled unstaking queues if large redemption volumes hit BENQI during a crash.

---

## 5. Comparison with Other Stablecoin Designs

| Mechanism | How It Handles Losses | Real-World Failure Mode |
| :--- | :--- | :--- |
| **MakerDAO / Sky** | Asynchronous Dutch liquidation auctions | Mempool congestion and auction slippage (e.g., Black Thursday $\$4.5M$ bad debt from 0-bid auctions) |
| **Liquity (LUSD/BOLD)** | Stability Pool absorption & user-set interest rates | Stability Pool can be depleted during long, deep bear markets |
| **Terra / UST** | Mint-and-burn arbitrage against endogenous LUNA | Reflexive death spiral when market confidence breaks |
| **anUSD** | Dual-tranche capital subordination + global scalar resets | Relies on sAVAX secondary liquidity, oracle uptime, and unbonding queues |

*(Scope Note: Our past research evaluated MakerDAO, Liquity, and Terra. We have not run formal models on Ethena USDe, so we do not make comparative claims about basis-trade synthetic tokens).*

---

## 6. Open Technical Questions (Where We Need BENQI Data)

There are two major areas where our models currently rely on simplified assumptions:

### 6.1 The sAVAX vs. AVAX Market Relationship
Our current models assume 1 sAVAX always trades at its exact fair redemption value. In reality:
1. **DEX Discounts:** During market panic, users often sell sAVAX on Pharaoh or LFJ instead of waiting. If sAVAX drops to a discount (e.g., $0.94\text{ AVAX}$), collateral value shrinks faster than spot AVAX.
2. **14-Day Unstaking Latency:** In a downward reset, senior holders receive physical sAVAX. If they all try to withdraw to native AVAX at once, BENQI’s immediate unstaking reserves could be exhausted, forcing users into the 14-day queue.
3. **Staking APR Fluctuations:** If network staking returns fall from 5% to 3%, senior coupon rates ($R, R'$) must adjust to prevent spread compression.

We need BENQI's real-world data on historical unbonding queue volumes and DEX liquidity depth to model these stress scenarios properly.

### 6.2 Parameter Selection (PSUU)
The numbers in this note:
* Reset barriers: $H_u = \$2.00$, $H_d = \$0.25$
* Senior coupon: $R = 7.3\%$
* anUSD base yield: $R' = 2.5\%$

These are **initial reference starting points**, not locked numbers. 

Choosing final parameters involves trade-offs:
* Barriers placed too close $\to$ resets happen too often, causing higher gas costs and rebalancing friction.
* Barriers placed too far apart $\to$ leverage swings too wide, increasing jump risk.

We intentionally have not locked these parameters because they should be calibrated together using BENQI's actual delegator retention and liquidity data.

---

## Closing Note

The core strength of anUSD is that it turns sAVAX into an active, productive reserve asset for a native dollar without exposing BENQI to liquidation auction failures.

Our current mathematical proofs and simulations provide a solid starting baseline. The next step is to sit down with BENQI's team, incorporate your actual market and unbonding queue data, and tune the parameters together.

***

*Bonding Curves Research Group (BCRG)*
