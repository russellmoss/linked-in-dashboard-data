# Weddings → RevOps Dashboard (Salesforce → Google Looker Studio) 💍📈

**Author:** **Russell Moss** — <a href="mailto:russell@mileaestatevineyard.com">[russell@mileaestatevineyard.com](mailto:russell@mileaestatevineyard.com)</a>
**Live Demo (Looker Studio):** <a href="https://lookerstudio.google.com/s/nRmeDByi1I4" target="_blank">[https://lookerstudio.google.com/s/nRmeDByi1I4](https://lookerstudio.google.com/s/nRmeDByi1I4)</a>

> *You can’t manage what you can’t measure.* This repo includes a **Salesforce-style CSV** and step-by-step instructions to build an interactive **Google Looker Studio** dashboard for a wedding-venue sales funnel — using the same **RevOps thinking** common in high-performing B2B revenue orgs.

**What you’ll build (free & repeatable):**

* **SLA Impact** (speed-to-lead vs booking rate) ⚡
* **Stage-to-Stage Conversion** (properly gated) 🔁
* **Stage Latency** medians & P90 (Call→Tour, Tour→Proposal, Sign→Deposit) ⏱️
* **Capacity Coverage** (Month × Day-of-Week heatmap with a target slider) 📅
* **Expected Value (EV)** for unsigned pipeline + **YTD Signed** and **YTD Deposits** KPIs 💰

---

## 📦 Repository Structure

```
.
├── salesforce_export_wedding_funnel_ASCII.csv   # Mock dataset (Salesforce-style export)
├── README.md                                    # This file
└── img/                                         # (optional) screenshots for the README
```

> **Data note:** This is mock data for education. No PII or real bookings.

---

## 🧠 Why this matters

We bring RevOps practices from B2B SaaS into a **brick-and-mortar wedding venue**. With **Salesforce** and a free tool like **Google Looker Studio** (or Tableau), you can see what truly moves bookings, coach to the numbers, and plan capacity with confidence.

---

## 🛠️ Quick Start (Looker Studio)

1. Open **Google Looker Studio** → **Blank report** → **Add data → Upload file → CSV** → select `salesforce_export_wedding_funnel_ASCII.csv`.
2. Add a **Date range control** to the page.

   * For *Funnel & SLA*, use **CreatedDate** as the date lens.
   * For *Capacity*, use **Wedding\_Date\_\_c** as the date lens.
3. Add slicers you care about (e.g., **LeadSource**, **Event\_Space\_\_c**, **Owner**).
4. Create the **calculated fields** below (copy/paste).

---

## 🧾 Dataset Overview

**Grain:** 1 row per inquiry/opportunity

**Key columns (typical Salesforce-style fields):**

* **Identity & meta:** `Id`, `Owner` (or `Owner.Name`), `Event_Space__c`, `LeadSource`
* **Timestamps (ISO strings):**
  `CreatedDate`, `First_Response__c`, `Call_Booked__c`, `Call_Held__c`,
  `Tour_Date__c`, `Proposal_Sent__c`, `Proposal_Signed__c`,
  `Deposit_Date__c`, `Wedding_Date__c`
* **Metrics:** `Amount`, `Deposit_Amount__c`, `Response_Minutes__c`

---

## 🧮 Calculated Fields (Looker Studio formulas we used)

### 1) Parse timestamps (DateTime)

```text
CreatedDate_dt      = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", CreatedDate)
First_Response_dt   = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", First_Response__c)
Call_Booked_dt      = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", Call_Booked__c)
Call_Held_dt        = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", Call_Held__c)
Tour_dt             = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", Tour_Date__c)
Proposal_Sent_dt    = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", Proposal_Sent__c)
Proposal_Signed_dt  = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", Proposal_Signed__c)
Deposit_dt          = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", Deposit_Date__c)
Wedding_dt          = PARSE_DATETIME("%Y-%m-%dT%H:%M:%SZ", Wedding_Date__c)
```

### 2) Stage flags (1/0) & SLA bucket

```text
Booked_Flag     = CASE WHEN Proposal_Signed_dt IS NOT NULL THEN 1 ELSE 0 END
Deposited_Flag  = CASE WHEN Deposit_dt        IS NOT NULL THEN 1 ELSE 0 END
Called_Flag     = CASE WHEN Call_Held_dt      IS NOT NULL THEN 1 ELSE 0 END
Toured_Flag     = CASE WHEN Tour_dt           IS NOT NULL THEN 1 ELSE 0 END
Proposed_Flag   = CASE WHEN Proposal_Sent_dt  IS NOT NULL THEN 1 ELSE 0 END
```

```text
Response_Bucket =
CASE
  WHEN Response_Minutes__c <= 15  THEN "0–15m"
  WHEN Response_Minutes__c <= 60  THEN "16–60m"
  WHEN Response_Minutes__c <= 180 THEN "61–180m"
  WHEN Response_Minutes__c <= 720 THEN "181–720m"
  ELSE ">12h"
END
```

### 3) Stage latencies (days) — used directly in percentiles

```text
Call_to_Tour_Days     = DATETIME_DIFF(Tour_dt,            Call_Held_dt,       DAY)
Tour_to_Proposal_Days = DATETIME_DIFF(Proposal_Sent_dt,   Tour_dt,            DAY)
Sign_to_Deposit_Days  = DATETIME_DIFF(Deposit_dt,         Proposal_Signed_dt, DAY)
```

**Chart-level percentile metrics:**

```text
Median_Call_to_Tour = PERCENTILE(Call_to_Tour_Days, 50)
Median_Tour_to_Prop = PERCENTILE(Tour_to_Proposal_Days, 50)
Median_Sign_to_Dep  = PERCENTILE(Sign_to_Deposit_Days, 50)

P90_Call_to_Tour    = PERCENTILE(Call_to_Tour_Days, 90)
P90_Tour_to_Prop    = PERCENTILE(Tour_to_Proposal_Days, 90)
P90_Sign_to_Dep     = PERCENTILE(Sign_to_Deposit_Days, 90)
```

### 4) Capacity coverage (Month × Day-of-Week)

```text
Wedding_Month_Label = FORMAT_DATETIME("%Y-%m", Wedding_dt)
Wedding_DOW         = FORMAT_DATETIME("%a",    Wedding_dt)   -- Sun..Sat
Wedding_DOW_Order =
CASE Wedding_DOW
  WHEN 'Sun' THEN 1 WHEN 'Mon' THEN 2 WHEN 'Tue' THEN 3
  WHEN 'Wed' THEN 4 WHEN 'Thu' THEN 5 WHEN 'Fri' THEN 6 WHEN 'Sat' THEN 7
END
Wedding_Date_Key    = SUBSTR(IFNULL(CONCAT('', Wedding_dt), ''), 1, 10)
```

**Parameter (slider):** <u>Target\_Per\_DOW</u> *(Number; Allowed values = Range `1–10`, Step `1`, Default `4`)*

```text
Coverage_Ratio  = COUNT_DISTINCT(Wedding_Date_Key) / Target_Per_DOW
Coverage_Capped =
CASE WHEN (COUNT_DISTINCT(Wedding_Date_Key) / Target_Per_DOW) > 1.5 THEN 1.5
     ELSE (COUNT_DISTINCT(Wedding_Date_Key) / Target_Per_DOW) END
```

### 5) EV for unsigned pipeline + YTD actuals

**Parameters (sliders):** <u>Tour\_Prob</u> (0–1), <u>Proposal\_Prob</u> (0–1) — set **Range** with step 0.05.

```text
Stage_Weighted_EV =
CASE
  WHEN Proposal_Signed_Flag = 1 THEN 0
  WHEN Proposed_Flag        = 1 THEN Amount * Proposal_Prob
  WHEN Toured_Flag          = 1 THEN Amount * Tour_Prob
  ELSE 0
END
```

```text
Signed_Value   = SUM(CASE WHEN Proposal_Signed_dt IS NOT NULL THEN Amount            ELSE 0 END)
Deposits_Value = SUM(CASE WHEN Deposit_dt         IS NOT NULL THEN Deposit_Amount__c ELSE 0 END)
```

---

## 📊 Pages to Build (and what each shows)

### 1) **Funnel & SLAs** *(Created Date lens)*

* **SLA Impact (bar):** `Response_Bucket` × `AVG(Booked_Flag)` → %
  *(Optional 2nd metric: `COUNT(*)` for sample size)*
* **Stage Conversion (one-row table):** chart-level metrics (examples)

  ```text
  Call Held→Tour:
  SUM(CASE WHEN Called_Flag=1 AND Toured_Flag=1 THEN 1 ELSE 0 END)
  / NULLIF(SUM(Called_Flag),0)

  Tour→Proposal:
  SUM(CASE WHEN Toured_Flag=1 AND Proposed_Flag=1 THEN 1 ELSE 0 END)
  / NULLIF(SUM(Toured_Flag),0)

  Proposal→Signed:
  SUM(CASE WHEN Proposed_Flag=1 AND Booked_Flag=1 THEN 1 ELSE 0 END)
  / NULLIF(SUM(Proposed_Flag),0)
  ```
* **Filters:** LeadSource, Event\_Space\_\_c, Owner; **Date:** `CreatedDate_dt`

### 2) **Latency** *(Created Date lens)*

* **Bar:** `Median_Call_to_Tour`, `Median_Tour_to_Prop`, `Median_Sign_to_Dep`
* **Small table:** `P90_Call_to_Tour`, `P90_Tour_to_Prop`, `P90_Sign_to_Dep`
* *Interpretation:* Medians show typical speed; **P90** reveals long-tail friction to fix.

### 3) **Booked Dates (Month × DOW)** *(Wedding Date lens)*

* **Pivot (counts):** Rows=`Wedding_Month_Label`; Columns=`Wedding_DOW_Order` then `Wedding_DOW`; Metric=`COUNT_DISTINCT(Wedding_Date_Key)`
* **Sort:** Columns by `Wedding_DOW_Order` Asc; **Filter:** exclude null `Wedding_DOW`
* **Date:** `Wedding_dt`

### 4) **Coverage Heatmap** *(Wedding Date lens)*

* Duplicate **Booked Dates** pivot; swap metric to `Coverage_Capped`
* **Conditional formatting:** 0.00 (red) → 1.00 (yellow) → 1.50 (green)
* **Control:** slider bound to **Target\_Per\_DOW** (label “Target slots per weekday”)

### 5) **EV vs Signed & Deposited** *(What-If + YTD)*

* **Scorecard:** `SUM(Stage_Weighted_EV)` (currency) — lens = `CreatedDate_dt`
* **Scorecard:** `Signed_Value` — **Date dim = Proposal\_Signed\_dt**, scope = **YTD**
* **Scorecard:** `Deposits_Value` — **Date dim = Deposit\_dt**, scope = **YTD**
* **Controls:** sliders bound to **Tour\_Prob** and **Proposal\_Prob**

---

## 🔌 Getting Data into the Dashboard (CSV today; direct or via warehouse tomorrow)

* **This repo:** uses a **CSV export** from Salesforce → Looker Studio.
* **Production options:**

  1. **Direct Salesforce → BI** (native connectors to Looker Studio/Tableau).
     *Fast setup; watch API limits & schema drift.*
  2. **Salesforce → Warehouse → BI** (Fivetran/Airbyte/Meltano → BigQuery/Snowflake/Redshift → Looker Studio/Tableau).
     *Snapshots, joins, governance — scales cleanly.*

**Minimal modeling needed:** one timestamp per stage, a first-response timestamp, `Amount`, and `Deposit_Amount__c`. Everything here drops in cleanly.

---

## 🧪 Troubleshooting

* **“Invalid argument type”** in a field → you likely used an aggregate (e.g., `SUM`) inside a **data-source** field. Move it to a **chart-level** metric or create **row-level flags** then aggregate.
* **Percentiles wrong/empty** → ensure you’re feeding numeric day counts (`DATETIME_DIFF(..., DAY)`), not text.
* **Slider doesn’t move** → set parameter to **Range** (min/max/step) and bind the control to the same **data source** as the chart.
* **Weekday order off** → include `Wedding_DOW_Order` as the first Columns field and sort Asc.

---

## 📜 License

* **Code & formulas:** MIT
* **Dataset:** CC BY 4.0 (*Attribution — “© Russell Moss, Milea Estate Vineyard”*)

---

## 📬 Contact

Questions, ideas, or adaptations for your funnel?
**Russell Moss** — <a href="mailto:russell@mileaestatevineyard.com">[russell@mileaestatevineyard.com](mailto:russell@mileaestatevineyard.com)</a>

If you ship your own variant (different stages, scoring, or a warehouse model), open an issue or PR — I’d love to compare notes.
