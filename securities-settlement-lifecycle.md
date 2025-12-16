
## **1. Securities Settlement Lifecycle**

When a trade happens (e.g., buying or selling bonds, equities), it doesn’t instantly result in ownership change and cash movement. The **settlement lifecycle** ensures that both happen correctly, in the right order, and with minimal risk.

**Main stages:**

1. **Trade Execution**

   * Buyer and seller agree on price, quantity, and terms.
   * Can be on an exchange, over-the-counter (OTC), or via a trading platform.

2. **Trade Capture & Confirmation**

   * Details are recorded in systems and matched with the counterparty’s records.
   * Trade matching eliminates discrepancies before settlement.

3. **Clearing**

   * The clearing system determines net obligations (how much securities and cash each party owes).
   * In some markets, a central counterparty (CCP) steps in to guarantee settlement.

4. **Settlement**

   * Actual delivery of securities from seller to buyer.
   * Cash transfer from buyer to seller.
   * Typically occurs on a set date, e.g., **T+2** (two business days after trade date).

5. **Post-Settlement Activities**

   * Record updates in custody accounts.
   * Corporate action entitlements, reporting, reconciliation.

---

## **2. The DvP Model (Delivery vs Payment)**

**Definition:**
DvP ensures that **securities are delivered if and only if payment is made**—both legs of the transaction occur **simultaneously**.

Think of it like exchanging money for goods in a store—**the cashier hands you the product at the same moment you hand over the cash**.

---

**BIS/CPSS (now CPMI) defines three main DvP models:**

* **Model 1:** Gross securities delivery vs gross funds transfer (each transaction settled individually in real time).
* **Model 2:** Gross securities delivery but net funds transfer at the end of the day.
* **Model 3:** Net delivery of both securities and funds.

---

## **3. Why DvP is Good for Securities Settlement**

* **Eliminates principal risk**
  Without DvP, there’s a risk that one party delivers securities but never receives payment (or vice versa). This is called *Herstatt risk* in FX markets or principal risk in securities.

* **Regulatory compliance**
  DvP is a **core principle** in the BIS Principles for Financial Market Infrastructures (PFMI), which many jurisdictions (including HKMA for CMU) follow.

* **Boosts market confidence**
  Knowing that settlement is atomic (both legs happen together) encourages participation and liquidity.

* **Reduces systemic risk**
  Failure of one transaction doesn’t cascade through the system because obligations are matched and settled in a controlled, synchronized manner.

---

**Example in CMU + CHATS context:**
If a bond trade settles in CMU, DvP links the securities ledger (CMU) with the cash leg in CHATS (HK’s RTGS system). Only when CHATS confirms cash payment does CMU release the securities to the buyer—removing the risk of partial completion.

---

**證券交收生命周期**的五個主要階段用中文可以這樣講：

1. **交易執行（Trade Execution）**
   買方與賣方就價格、數量、交收日等條款達成協議，可通過交易所、場外交易（OTC）或電子平台完成。

2. **交易確認（Trade Confirmation / Matching）**
   雙方對交易細節進行記錄、比對及確認，確保資料一致，避免交收出錯。

3. **清算（Clearing）**
   計算並確定各方應收應付的證券和資金數量，有些市場會由中央對手方（CCP）介入，承擔交收擔保。

4. **交收（Settlement）**
   實際完成證券與資金的交換。通常會用**付款交割制（DvP, Delivery vs Payment）**，確保證券和資金同時交收，避免本金風險。

5. **交收後處理（Post-Settlement）**
   更新託管帳戶記錄、處理公司行動（如派息、配股）、生成報表以及進行對賬等工作。

