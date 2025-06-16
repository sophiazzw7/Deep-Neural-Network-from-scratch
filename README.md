Here's a concise and well-organized **meeting notes summary** based on your detailed conversation:

---

### 📝 **Meeting Notes – OGM Monitoring & Recalibration Discussion**

**Date:** \[Insert Date]
**Participants:** \[Insert Names if needed]

---

#### 🔍 **1. PSI Population Clarification**

* There was confusion regarding whether PSI metrics in the OGM report were calculated using **full population** or **auto-approved applicants**.
* Action: Team to clarify and document the population used for both **model output PSI** and **feature distribution PSI** in the next report.
* Noted that PSI based on full population may show significant breach due to marketing-driven low-quality applicants.

---

#### 🧠 **2. Sensitivity Analysis Concerns**

* Current PSI-based sensitivity test shifts the model score ±20% to assess PSI response.
* Consensus: This method does **not provide meaningful insights** and may be misleading, as shifting scores inherently increases PSI.
* Suggested alternative: Test **model sensitivity to individual feature shifts** using **AUC or KS** instead.
* Decision: **Sensitivity analysis will no longer be included** in regular OGM reports unless formally required and justified.

---

#### 📈 **3. KS and AUC Thresholds**

* Concern raised that **KS threshold** was recently updated based on deteriorated Q4 2024 performance.
* Recommendation: Thresholds should be based on **development data** or a historically strong period, not a declining one.
* Action: Revisit and update KS threshold logic; include **KS thresholds and rank ordering tables** in future OGM reports alongside AUC.

---

#### ⚠️ **4. Ongoing Breaches and Recalibration**

* Noted breaches in AUC and PSI in both 2024 Q2 and Q4 reports.
* Per OGM framework, **Q2 breach should have already triggered a recalibration action plan.**
* Recalibration contract has been signed; however, timeline and execution details remain unclear.
* Action: Follow up with model owner to confirm:

  * Whether the original recalibration plan (including segmentation and feature changes) is still being followed.
  * Expected **start date**, **timeline**, and **scope** (refit vs. rebuild).

---

#### 📊 **5. Early Warning Indicator Clarification**

* Misalignment noted in how **“early warning indicator”** is being interpreted.
* Clarified that:

  * **Performance metrics** = AUC, KS (assess effectiveness)
  * **Early warning indicators** = PSI (assess population or feature drift)
* Agreement: Early reports may rely only on early warning indicators (e.g., PSI), with performance metrics added once enough data (e.g., 18 months post-implementation) is available.

---

#### ✅ **6. Action Plans and Documentation**

* Reconfirmed that an **action plan has been submitted** previously.
* Need to verify if the plan and **timeline** are being followed as described.
* Request: Ensure evidence of ongoing communication and discussions (e.g., email trails) regarding the recalibration is documented for audit purposes.

---

#### ⚖️ **7. Fairness / Disparity Assessment**

* Internal fairness review flagged **discrimination concerns** in a few segments for the AI/ML model (e.g., white vs. Black/Hispanic).
* Traditional models reportedly passed the review.
* No current directive to halt model use; however, rebuild efforts are ongoing.
* Action: Monitor fairness trends in future validations and stay aligned with evolving expectations.

---

### 📌 **Next Steps**

* Clarify PSI population basis in all future reports.
* Remove PSI-based sensitivity test; propose AUC/KS-based sensitivity if needed.
* Confirm KS threshold setting and update with rationale.
* Follow up on recalibration scope, segmentation changes, and timelines.
* Ensure early warning indicators and performance metrics are correctly classified and scheduled.
* Submit and track evidence of ongoing remediation activities per action plans.

---

Let me know if you'd like a version in email format or editable Word format.
