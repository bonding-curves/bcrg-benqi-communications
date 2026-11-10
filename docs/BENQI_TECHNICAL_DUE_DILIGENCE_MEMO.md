# anUSD: Mechanism, Yield Allocation, and Risk Considerations

**To:** Hansen Niu, BENQI Protocol & Risk Engineering Team  
**From:** Bonding Curves Research Group (BCRG)  
**Date:** November 10, 2026  
**Status:** Working Draft for Technical Discussion

---

## 1. Purpose

This note provides a simplified explanation of the proposed Avalanche Native USD (anUSD) mechanism and responds to the questions raised by Hansen.

It covers four areas:

1. How the two-tranche mechanism works and how the $2\times$ leverage is maintained.
2. How sAVAX staking rewards are intended to flow through the different tokens.
3. How the system responds to large AVAX price movements.
4. How the design compares with existing stablecoin mechanisms.

It also identifies the assumptions that still need to be tested with BENQI data.

The parameters used below are illustrative. They should not be treated as final deployment parameters.

---

## 2. How the Mechanism Works

### 2.1 Two tranches backed by sAVAX

The basic idea is to deposit sAVAX into a vault and divide the resulting economic exposure into two tranches.

**Senior tranche: anBOND**

This is the senior claim on the collateral.

It is designed to preserve principal and receive a coupon at rate $R$. Our initial example uses $R = 7.3\%$.

**Junior tranche: anAVAX**

This is the residual equity position.

It absorbs changes in the value of the underlying AVAX collateral after accounting for the senior claim. It therefore carries more risk but also receives more upside when AVAX appreciates.

The basic structure is:

* Users deposit sAVAX.
* The vault holds the collateral.
* The senior tranche receives its contractual claim.
* The junior tranche absorbs the remaining gains or losses.

At the start of an epoch, the target allocation is:

| Component | Value | Notes |
| :--- | ---: | :--- |
| **Total sAVAX collateral** | **\$2.00** | Deposited in custodian vault |
| **Senior claim (anBOND)** | \$1.00 | Priority claim |
| **Junior equity (anAVAX)** | \$1.00 | Residual equity |

There are therefore \$2 of collateral for every \$1 of junior equity.

This gives the junior position $2\times$ leverage at the start of the epoch.

### 2.2 How anUSD and anYIELD fit in

The proposal introduces a second layer of tokens.

The senior claim is intended to provide two different economic exposures:

* **anUSD:** A dollar-denominated stablecoin targeting \$1.00.
* **anYIELD:** A token representing the additional yield allocation.

The intended distinction is that anUSD provides the stable monetary unit, while anYIELD provides exposure to the remaining yield.

The current design specifies a conversion rule:

* **Mint:** burn two units of Token A to receive one anUSD and one anYIELD.
* **Redeem:** burn one anUSD and one anYIELD to receive two units of Token A.

We should confirm the precise definition and denomination of Token A, including whether the ratio refers to token units or dollar value. This is important for establishing that the conversion preserves value.

### 2.3 How the 2× leverage changes with AVAX

The junior tranche is calculated as the value of the collateral minus the senior claim.

Using the current model:

$$V_B(t) = 2 S_t - V_A(t)$$

Where:

* $S_t$ is the AVAX price index, normalised to $1.0$ at the beginning of an epoch.
* $V_A(t)$ is the senior claim.
* $V_B(t)$ is the junior tranche value.

Consider two examples, ignoring coupon accrual for simplicity.

**Scenario A: AVAX increases by 50%.**

The collateral increases from \$2.00 to \$3.00.

The senior claim remains \$1.00, leaving \$2.00 for the junior tranche.

The junior position has therefore doubled in value, while leverage falls from $2.0\times$ to $1.5\times$.

**Scenario B: AVAX decreases by 37.5%.**

The collateral falls from \$2.00 to \$1.25.

After the \$1.00 senior claim, only \$0.25 remains for the junior tranche.

Leverage consequently increases from $2.0\times$ to $5.0\times$.

The important point is that leverage is not kept constant through ordinary price movements. It changes with the value of the junior equity.

The proposal uses periodic resets to bring the system back to its target leverage.

### 2.4 Reset mechanism

Rather than liquidating individual positions through auctions, the proposal uses protocol-level resets.

There are two proposed triggers.

**Upward reset**

When anAVAX reaches \$2.00:

* The system recognises the gains in the junior position.
* The design proposes distributing gains in sAVAX.
* A 2-for-1 share split is used to bring the token price back towards \$1.00.

**Downward reset**

When anAVAX falls to \$0.25:

* The system proposes paying \$0.75 of the original senior claim to senior holders in sAVAX.
* The remaining senior and junior positions are consolidated through a 4-for-1 reverse split.
* The objective is to reduce outstanding senior liabilities and restore the target leverage.

The intention is to avoid relying on a large number of individual liquidation transactions.

The reset logic is intended to require a fixed amount of computation. However, execution still depends on the oracle, transaction inclusion and the relevant on-chain trigger mechanism.

We should provide a before-and-after balance sheet for both resets, particularly to establish how the upward profit distribution and share split work together.

---

## 3. How Staking Yield Is Distributed

The collateral consists of sAVAX, which generates staking rewards.

The system has two dollars of sAVAX for every dollar of initial senior exposure. This means the gross staking yield is earned on twice the collateral value.

For illustration, assuming sAVAX earns $4.5\% - 5.5\%$:

$$2 \times (4.5\% - 5.5\%) = 9.0\% - 11.0\%$$

This represents gross staking rewards relative to the initial \$1 senior exposure, before protocol costs and other adjustments.

The proposed yield allocation has three main components.

**1. Senior allocation**

The initial model uses a senior coupon $R$ of $7.3\%$.

**2. anUSD and anYIELD**

The proposal uses anUSD base yield $R'$ of $2.5\%$.

The remaining yield is intended to flow to anYIELD. The current formula specifies:

$$R_{\text{anYIELD}} = 2R - R'$$

Using the illustrative rates:

$$2(7.3\%) - 2.5\% = 12.1\%$$

These rates are provisional. We need to reconcile the formula with actual staking rewards and confirm the notional against which each percentage is calculated.

**3. Ecosystem allocation**

The proposal also directs residual protocol surplus towards the Avalanche ecosystem.

The current allocation proposal is:

| Allocation | Proposed use |
| :--- | :--- |
| 65% | AVAX buyback and burn |
| 20%–45% | Additional validator staking rewards |
| 15% | Avalanche L1 ecosystem grants |

The validator allocation can vary depending on network conditions, particularly when AVAX trades below its 90-day moving average.

The percentage allocation needs a final definition. The current ranges can add up to more than 100%, so we need to specify whether the validator allocation comes out of the buyback allocation or is calculated separately.

---

## 4. What Happens During a Large AVAX Price Decline?

We consider two scenarios.

### 4.1 Gradual decline over 30 days

Suppose AVAX loses 50%–60% of its value over a month.

The proposed mechanism is designed to respond through its lower reset boundary.

The intended sequence is:

1. The junior tranche absorbs the initial loss.
2. When anAVAX reaches the lower boundary of \$0.25, the downward reset is triggered.
3. The senior claim is amortised, with \$0.75 proposed to be returned to senior holders.
4. The remaining positions are consolidated.
5. The recapitalised system continues under the new capital base.

If AVAX continues falling, the same mechanism may need to be applied again.

The objective is to reduce senior liabilities before losses reach the senior tranche.

### 4.2 Sudden price decline (flash crash)

A more difficult scenario is an immediate price decline, where the system does not have time to execute an intermediate reset.

Our current analysis reports the following theoretical bounds:

| Starting state | Reported maximum instantaneous decline |
| :--- | ---: |
| Initial par state | **75%** |
| Lower reset boundary | **60%** |

The current cadCAD simulations also report no anUSD haircuts across 500 jump-diffusion paths.

However, these results need to be reconciled with the collateral equations and reset accounting before we present them as established risk bounds.

In particular, under the simplified 2:1 model, with a \$1 senior claim and \$2 of initial collateral:

$$2 S_t \ge 1$$

This gives a 50% decline in collateral price as the maximum loss consistent with full senior coverage, assuming no additional reserves or protection.

At the lower reset boundary, the stated \$0.75 amortisation leaves \$0.50 of collateral against a \$0.25 senior claim. This simple balance sheet implies protection against a further 50% decline, rather than 60%.

These calculations do not account for any additional protection that may be specified elsewhere in the mechanism.

We should therefore include the detailed proof and the exact assumptions behind the reported 75% and 60% results before treating them as final.

### 4.3 Simulation results

The current simulation uses:

* 2,140 days of historical Avalanche price data.
* 500 jump-diffusion price paths.
* A 365-day simulation horizon for the jump scenarios.

No senior haircut occurred in the reported runs.

This is an encouraging result for the tested parameter set. It should not be interpreted as evidence that the real system has zero haircut probability.

---

## 5. Comparison with Existing Stablecoin Designs

The main difference between anUSD and existing designs is how losses are handled.

| Design | Main mechanism | Main risk or limitation |
| :--- | :--- | :--- |
| **MakerDAO / Sky** | Collateral-backed debt with liquidation auctions | Auction execution and liquidity during market stress |
| **Liquity** | Stability Pool and redistribution | Stability Pool capacity and redistribution during large liquidations |
| **Terra / UST** | Endogenous mint-and-burn relationship with LUNA | Reflexive supply expansion and loss of confidence |
| **anUSD** | sAVAX-backed senior and junior tranches with global resets | sAVAX liquidity, oracle execution and reset assumptions |

The proposed anUSD mechanism takes a different approach.

Instead of depending primarily on individual liquidation auctions, it attempts to manage losses through the relationship between the senior and junior tranches and predefined reset conditions.

This could reduce some of the operational complexity associated with liquidation auctions.

However, it does not remove execution risk. The mechanism still depends on reliable collateral valuation, sufficient market liquidity and successful on-chain execution.

Our previous post-mortem work has covered MakerDAO/Sky, Terra and Liquity V1/V2.

We have not yet conducted equivalent formal modelling or simulation work on Ethena (USDe). We therefore do not make comparative risk claims about that design.

---

## 6. What We Still Need to Model

There are two main areas where we would like to work with BENQI.

### 6.1 sAVAX and AVAX market relationship

Our current model assumes that sAVAX always tracks its theoretical redemption value.

This is a simplifying assumption.

In practice, the market price of sAVAX can diverge from its underlying AVAX value, particularly during periods of market stress.

There are three issues to consider.

**DEX liquidity and discounts**

During a sell-off, users may sell sAVAX on LFJ or Pharaoh rather than wait for redemption.

For example, sAVAX could trade at $0.94\text{ AVAX}$ while its theoretical redemption value remains higher.

This creates a difference between the market value of the collateral and the value assumed by the model.

**Unstaking and redemption delays**

The model also needs to account for the time required to convert sAVAX back into AVAX.

The current analysis uses a 14-day unstaking assumption.

During a large redemption event, available liquidity may be insufficient to meet immediate demand. This could affect both collateral market prices and the ability of holders to access native AVAX.

**Staking yield changes**

Staking rewards can change over time.

For example, a decline in staking APY from $5\%$ to $3\%$ would affect the amount of yield available to meet the senior coupon and other allocations.

We would like to incorporate historical sAVAX/AVAX market prices, trading volumes, redemption activity and unbonding queue data into the simulation.

### 6.2 Parameter selection under uncertainty (PSUU)

The current parameters are starting points:

| Parameter | Initial value |
| :--- | ---: |
| Upper reset boundary, $H_u$ | \$2.00 |
| Lower reset boundary, $H_d$ | \$0.25 |
| Senior coupon, $R$ | 7.3% |
| anUSD base yield, $R'$ | 2.5% |

We have not completed the full parameter selection process.

The main trade-off is straightforward:

* Reset boundaries that are too close together may lead to frequent resets and additional execution costs.
* Boundaries that are too far apart allow larger changes in leverage before the system responds.

The optimal values will depend on BENQI-specific data, including sAVAX liquidity, redemption behaviour, staking yield and market depth.

We would prefer to establish these parameters jointly rather than fix them using assumptions that have not yet been tested against BENQI data.

---

## 7. Proposed Next Steps

We suggest three areas of collaboration:

1. **Review the mechanism:** Confirm that the token structure, mint/redeem rules, senior coupon and reset accounting reflect the intended BENQI architecture. We would particularly like to reconcile the crash bounds and yield waterfall.
2. **Incorporate BENQI data:** Use historical sAVAX/AVAX market data, DEX liquidity, staking rewards and unstaking queue history to improve the cadCAD model.
3. **Joint parameter selection:** Run the parameter selection process using realistic market and redemption scenarios. The objective would be to identify a suitable initial parameter set for a Fuji testnet deployment.

---

## Closing

The central idea behind anUSD is to use a senior/junior structure to separate stable dollar exposure from leveraged AVAX exposure, with protocol-level resets to manage changes in collateral value.

The current model provides a starting point for analysing this mechanism. The next step is to bring in BENQI's actual sAVAX market and redemption data, reconcile the remaining accounting and risk calculations, and determine which parameters are suitable for deployment.

We would welcome BENQI's feedback on the mechanism and the outstanding assumptions.

***

*Bonding Curves Research Group (BCRG)*
