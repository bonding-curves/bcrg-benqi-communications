# BCRG <> BENQI Protocol Communications & Technical Due Diligence Repository

This repository houses formal technical communications, due diligence memoranda, mathematical models, and collaboration working papers between the **Bonding Curves Research Group (BCRG)** and the **BENQI Protocol & Risk Engineering Team** regarding the **Avalanche Native USD (anUSD)** architecture.

---

## 🏛️ Scope & Epistemic Boundaries

This repository is dedicated exclusively to client and partner technical dialogue surrounding the anUSD dual-class securitization model backed by liquid-staked AVAX ($sAVAX$).

> [!IMPORTANT]
> **Separation of Concerns:**  
> This workspace is maintained as an independent technical collaboration between BCRG and BENQI. It is strictly segregated from Avalanche Foundation grant research repositories and formal foundation deliverables (`avalanche-phase-ii-*` / `ext-bcrg-validator-research`).

---

## 📄 Active Documents

| Document | Classification | Subject | Date |
| :--- | :--- | :--- | :--- |
| [`docs/BENQI_TECHNICAL_DUE_DILIGENCE_MEMO.md`](docs/BENQI_TECHNICAL_DUE_DILIGENCE_MEMO.md) | **Technical Due Diligence Memo** | Line-by-line responses to Hansen's questions regarding dynamic 2x leverage, sAVAX yield waterfall, 50–60% AVAX crash bounds (Theorem 1), comparative design taxonomy, and the $sAVAX \leftrightarrow AVAX$ open research frontier. | Nov 10, 2026 |

---

## 🔬 Core Architectural Themes Covered

1. **Dual-Class Tranching:** Senior Class A (anBOND) and Junior Class B (anAVAX) with strict 2:1 secondary tranching into anUSD (Class A$'$) and anYIELD (Class B$'$).
2. **Autonomous $O(1)$ Resets:** Rebalancing leverage to $2.0\times$ at upper ($H_u = \$2.00$) and lower ($H_d = \$0.25$) barriers without asynchronous debt auctions.
3. **Crash Tolerance Bounds:** Analytical **Theorem 1** proving zero haircut tolerance up to a **−60% flash crash from barrier** and **−75% from par**, backed by cadCAD digital twin validation (0.00% haircut over 500 Kou jump paths).
4. **Open Research Frontier:** Explicit acknowledgment that the empirical dynamic coupling of $sAVAX \leftrightarrow AVAX$ (de-peg risk, liquidity premiums, unbonding queue delays) requires joint co-modeling with BENQI.
