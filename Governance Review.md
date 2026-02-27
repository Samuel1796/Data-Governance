# QuickLoan Mobile - Ethical Data Governance Review


## Deliverable 1: Governance Review Card

### 1. Data Quality Risk

| Field | Detail |
|---|---|
| **Issue** | Customer data entering the pipeline is incomplete and inconsistently formatted (e.g., missing income fields, varied phone number formats, blank addresses). The Preprocessing Service has no defined handling rules for these gaps. |
| **Impact** | Incomplete records feed directly into the ML model, producing unreliable loan scores. Bad input = bad decisions - applicants may be wrongly rejected or approved. |
| **Suggested Fix** | Add a **data validation layer** at the Preprocessing Service (Step 5). Define mandatory fields, enforce standard formats, and flag or quarantine incomplete records before they reach the ML model. Introduce a Data Quality dashboard to track completeness rates weekly. |

---

### 2. Legal & Compliance Risk

| Field | Detail |
|---|---|
| **Data Classification** | **Sensitive** - the pipeline handles National ID numbers, phone contacts, GPS location, financial history, and device logs. All are personally identifiable and financially sensitive under Ghana's DPA. |
| **Issue** | There is **no consent capture** between the API Gateway (Step 2) and the Raw Data DB (Step 3). The app also collects far more data than needed - contact lists, GPS, device logs - with no stated purpose for each. This directly violates Ghana's Data Protection Act, 2012 (Act 843). |
| **Impact** | ShopGhana is processing sensitive PII without a lawful basis. Under Act 843, this exposes the company to investigation by the Ghana Data Protection Commission (GDPC), fines, and reputational damage. Customers have no idea what is being collected or why. |
| **Suggested Fix** | Insert a **Consent & Compliance Gateway** between Steps 2 and 3. This module must: (1) present a clear, plain-language consent screen before data is stored, (2) log consent with timestamp and version, (3) apply **data minimisation** - strip any fields not required for a loan decision (contact list, GPS, device logs), and (4) enforce a **retention policy** (e.g., auto-delete raw data after 90 days). |

---

### 3. Bias & Fairness Risk

| Field | Detail |
|---|---|
| **Source of Bias** | The ML model (Step 6) is trained on historical loan data. If past approvals were skewed - e.g., fewer approvals for users in certain regions, with certain name patterns, or lower device tiers - the model learns and **repeats those biases automatically**. The Decision Service (Step 7) also has no transparency logging, meaning biased decisions are invisible. |
| **Impact** | Certain demographic groups (by gender, region, or income bracket) may be systematically denied loans at higher rates - not because of creditworthiness, but because of patterns in historical data. This is discriminatory and unethical, even if unintentional. |
| **Suggested Fix** | (1) Audit training data for demographic imbalance before the next model retrain. (2) Add **transparency logging** at the Decision Service - every auto-decision must record the top contributing features and the applicant's demographic segment. (3) Run monthly **Approval Rate Disparity checks** across demographic groups (see Deliverable 1, Section 4). |

---

### 4. Storytelling / Reporting Metric

| Field | Detail |
|---|---|
| **Metric Name** | **Loan Approval Rate by Demographic Group** |
| **Definition** | The percentage of loan applications approved, broken down by demographic segment (e.g., gender, region, age group) - calculated monthly from the Decision Service logs. Formula: `(Approvals in segment ÷ Total applications in segment) × 100` |
| **Visualization Type** | **Grouped Bar Chart** - one bar per demographic group per month, making disparities immediately visible at a glance |
| **Why It Matters** | If one group is approved at 70% and another at 30% for similar credit profiles, the model has a fairness problem - and this metric surfaces that before it becomes a legal or reputational crisis. |

---

---

## Deliverable 2: Corrected Data Flow Diagram

The diagram below shows the corrected pipeline. Flawed steps are marked with numbered corrections explained in the annotations below.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  QUICKLOAN CORRECTED DATA FLOW                          │
└─────────────────────────────────────────────────────────────────────────┘

 ┌──────────────────┐         ┌───────────────────┐
 │  1. User Mobile  │─[¹]────▶│  2. API Gateway   │
 │  App             │         │                   │
 └──────────────────┘         └────────┬──────────┘
                                       │
                                      [²]
                                       ▼
                              ┌─────────────────────────┐
                              │   Consent & Compliance │
                              │     Gateway (NEW)        │
                              │  - Capture explicit      │
                              │    consent               │
                              │  - Strip non-essential   │
                              │    data fields           │
                              └────────────┬────────────┘
                                           │
                                          [³]
                                           ▼
                              ┌─────────────────────────┐
                              │  3. Raw Data DB         │
                              │     'AllEvents'          │
                              │   Classified: Sensitive │
                              │   Retention: 90 days   │
                              └────────────┬────────────┘
                                           │
                                          [⁴]
                                           ▼
                              ┌─────────────────────────┐
                              │  5. Preprocessing        │
                              │     Service              │
                              │   Validation rules      │
                              │   Quarantine bad data   │
                              └────────────┬────────────┘
                                           │
                                           ▼
                              ┌─────────────────────────┐
                              │  6. LoanScore ML Model  │
                              └────────────┬────────────┘
                                           │
                                          [⁵]
                                           ▼
                              ┌─────────────────────────┐
                              │  7. Decision Service    │
                              │   Transparency logging │
                              │   Record: features     │
                              │    used + outcome       │
                              └────┬───────────┬────────┘
                                   │           │
                                   ▼           ▼
                    ┌──────────────────┐  ┌──────────────────┐
                    │  8. Customer     │  │  9. Analytics DB │
                    │  Notification    │  │   PII Masked    │[⁶]
                    │  (SMS/Email)     │  │   Anonymised    │
                    └──────────────────┘  └──────────────────┘
```

### Correction Annotations

**[1] - Enforce Data Minimisation at the Mobile App**
*What:* Remove collection of contact list, GPS location, and device logs from the app.
*Why:* These are not needed to assess creditworthiness. Collecting them violates Ghana DPA Act 843's data minimisation principle (§17) and exposes QuickLoan to regulatory risk.

**[2] - Insert a Consent & Compliance Gateway**
*What:* Add a new consent module between the API Gateway and the Raw Data DB that captures explicit, informed user consent before any data is stored.
*Why:* Without this, QuickLoan is processing sensitive PII with no lawful basis - a direct violation of Act 843. Consent must be logged with a timestamp and version for audit purposes.

**[3] - Classify and Apply Retention Policy to Raw Data DB**
*What:* Tag all data in the Raw Data DB as **Sensitive** (per data classification standards) and enforce a 90-day automatic deletion policy on raw records.
*Why:* Unclassified data is treated inconsistently. Without a retention limit, QuickLoan holds sensitive financial data indefinitely, increasing breach risk and non-compliance exposure.

**[4] - Add Validation Rules at Preprocessing Service**
*What:* Introduce input validation - mandatory fields, format checks, and quarantine rules for incomplete records before data reaches the ML model.
*Why:* Garbage in = garbage out. Incomplete or malformed data produces unreliable loan scores, leading to unfair or inaccurate decisions.

**[5] - Enable Transparency Logging at Decision Service**
*What:* Log every automated decision with the top model features that influenced it, the outcome, and a demographic segment tag.
*Why:* Without logging, biased decisions are invisible and unauditable. Logging enables fairness monitoring and gives customers a basis for appeal - required for ethical and legally defensible automated decision-making.

**[6] - Mask/Anonymise PII in the Analytics DB**
*What:* Apply data masking or pseudonymisation to all PII fields before data enters the Analytics DB.
*Why:* Analytics workloads don't need raw names, phone numbers, or IDs. Storing unmasked PII in analytics systems unnecessarily multiplies breach surface area and violates the principle of data minimisation.

---

---

## Deliverable 3: Summary of Review Process

My review of QuickLoan's data pipeline used two core frameworks: **Data Lifecycle Management** and **Data Classification**, applied at each stage from collection to storage to decision-making.

Starting at the collection point, I applied the **Data Lifecycle lens** - asking at every stage: *what data is being collected, why, and what happens to it next?* This immediately surfaced the excessive collection problem at Step 1. Contact lists and GPS location have no logical role in a loan scoring decision, and their presence signals that the pipeline was built for growth speed, not governance.

Moving to **Data Classification**, I assessed the sensitivity of the data flowing through the pipeline. Everything in this system - financial history, phone numbers, location - qualifies as **Sensitive PII** under Ghana's DPA. Yet the pipeline treated it without any classification-based controls: no masking in the Analytics DB, no retention limits in the Raw Data DB, no consent gate before storage. Classification makes risk visible - once you label data as Sensitive, the absence of controls becomes immediately obvious.

The bias risk emerged from combining lifecycle and classification thinking with an **ethical fairness check**. Historical training data is itself a data artifact with a lifecycle - and if that data was generated by a biased system, the bias is baked into the model permanently unless actively audited.

My proposed metric - **Loan Approval Rate by Demographic Group** - directly addresses the transparency gap. By breaking down approval rates monthly across gender, region, and age, this metric turns an invisible algorithmic problem into a visible, trackable number. A grouped bar chart makes disparities easy to spot without needing technical expertise. Governance isn't just about fixing problems - it's about building the monitoring systems that catch problems before they cause harm. This metric is that early warning system.

