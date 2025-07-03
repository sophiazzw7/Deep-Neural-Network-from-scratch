Here's your revised email, fully incorporating your requests, providing specific references, detailed examples, and clarity on volatility sources:

---

**Subject:** Proposal to Adjust KS Thresholds Using Pooled 3σ Control Limits

Hi \<Name>,

I've carefully reviewed the current KS threshold methodology and propose a minor adjustment that retains the bootstrap framework while improving threshold stability and interpretability.

---

### Current Approach Overview

The existing method for setting thresholds involves:

1. Drawing 1,000 bootstrap samples per quarter (Q1–Q4 plus "All Quarters").
2. Computing mean (μ) and standard deviation (σ) for each quarter.
3. Defining thresholds: Red = μ − 4σ; Green = μ − 3σ.
4. Selecting the **minimum** of the five quarterly thresholds.

This approach performs well for AUC since its variance is relatively low (\~0.02). However, it becomes problematic for KS, as KS metrics show substantially higher volatility (σ ranges \~0.035 to 0.09). Specifically, Q2 had notably fewer default events, causing a large spike in standard deviation and significantly lowering the final thresholds.

For instance:

| Segment            | Q4 KS | Current Red Threshold | Performance Decline Needed for Red Flag |
| ------------------ | ----- | --------------------- | --------------------------------------- |
| Auto               | 0.380 | 0.1616 (set by Q2)    | Approximately 57% decline               |
| Debt Consolidation | 0.277 | 0.1310 (set by Q2)    | Approximately 59% decline               |
| Home Improvement   | 0.282 | 0.2475                | Approximately 44% decline               |
| Others             | 0.270 | 0.1540 (set by Q2)    | Approximately 60% decline               |

In particular, the Auto, Debt Consolidation, and Others segments would need to lose over half of their discriminatory power before triggering a red alert, significantly diminishing the practical effectiveness of KS monitoring.

---

### Proposed Modification

I suggest a minor yet impactful refinement:

* Pool all funded loans across quarters first ("All Quarters") to reduce sensitivity to any single volatile quarter.
* Replace the current 4σ (red) and 3σ (green) thresholds with **standard 3σ (red) and 2σ (yellow)** control limits.
* Eliminate the "minimum-of-five" quarterly threshold selection, since pooling sufficiently addresses quarterly volatility.

Applying the pooled approach using the values from Table 18-1 (All Quarters):

| Segment            | μ (pooled) | σ (pooled) | Red (μ − 3σ) | Yellow (μ − 2σ) |
| ------------------ | ---------- | ---------- | ------------ | --------------- |
| Auto               | 0.3970     | 0.0350     | 0.2920       | 0.3270          |
| Debt Consolidation | 0.3178     | 0.0428     | 0.1894       | 0.2322          |
| Home Improvement   | 0.4412     | 0.0527     | 0.2831       | 0.3358          |
| Others             | 0.3841     | 0.0288     | 0.2977       | 0.3265          |

---

### Rationale and References for 3σ Control Bands

* The **3σ limit** is a standard in statistical process control (SPC), originating from Shewhart's control chart methodology and widely endorsed across industries (see Montgomery, *Introduction to Statistical Quality Control*, 2019).
* Three-sigma limits correspond to a 99.7% confidence interval assuming normality, widely accepted as the industry norm for identifying significant deviations that warrant review or intervention.
* Adopting 3σ ensures thresholds remain actionable and appropriately sensitive to genuine deterioration in model performance.

---

### Why Q2 Caused Excessive Threshold Volatility

Quarter 2 specifically had fewer defaults, causing unusually large standard deviations:

* **Auto segment:** Q2 KS σ = 0.0773 (more than double the pooled σ of 0.0350).
* **Debt Consolidation segment:** Q2 KS σ = 0.0923 (more than double the pooled σ of 0.0428).

This unusually high variance in Q2 drove the calculated red thresholds down significantly (0.1616 for Auto and 0.1310 for Debt Consolidation), which is much lower than practically useful or plausible given historical performance.

---

### Next Steps

If you're comfortable with this approach, I'd be happy to update our threshold tables accordingly and share a side-by-side comparison of recent quarterly performance.

Thank you, and let me know if you have any questions or would like to discuss further.

Best regards,
\<Your Name>

---

**Reference:**

* Montgomery, D.C. (2019). *Introduction to Statistical Quality Control* (8th ed.). Wiley. (Chapter 5: "Methods and Philosophy of Statistical Process Control")
