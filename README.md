Here is a professional report section focused **only on the production period**, incorporating both the **OGM results** and the **Wilcoxon Rank-Sum Test** results based on the images you provided:

---

### **9.2 Back Testing — Production Period**

During the production period, model performance was monitored through quarterly Ongoing Monitoring (OGM) reports, with a focus on both calibration accuracy and discriminatory power. Given the limited number of defaults observed in each quarter, the assessment emphasizes directional validation rather than formal threshold conclusions.

#### **OGM-Based Monitoring**

The OGM reports for 202306, 202309, and 202312 presented limited default counts for PDA (4, 6, and 8 respectively) and moderately higher counts for PDB (12, 21, and 26 respectively). These are below the standard 30-default threshold required for formal Gini evaluation, and thus Gini values reported (PDA: 0.62, 0.50, 0.47) are considered early readings. While indicative of some discriminatory power, no conclusive judgment is drawn per model risk policy.

Despite these limitations, there is no observed systematic over- or under-estimation of predicted PDs in production, and the models maintain relative consistency in risk differentiation across the portfolio. PDA continues to be calibrated conservatively due to the absence of reject inference, consistent with development design.

#### **Wilcoxon Rank-Sum Test**

To supplement limited Gini analysis, the Wilcoxon Rank-Sum Test was applied to production data, assessing whether default rates increase monotonically across risk score buckets (i.e., whether higher-risk classes experience higher default incidence).

* For PDA and PDB, the Wilcoxon test was run on three variants: `Cal_RR`, `Kappa_RR`, and `Final_RR`.
* Across all variants, both models demonstrated **increasing default rates across the final four buckets**, satisfying the Wilcoxon test condition for discriminatory validity.
* Default counts in the higher-risk buckets were generally sufficient to support the directional conclusions of the test, particularly for PDB (`DefaultCount`: 52, 67, 52 in Classes 3–5 in `Final_RR`).
* As per the methodology, the model passes the Wilcoxon test when default rates across Classes 2 to 5 are in non-decreasing order. This condition was met, indicating **preserved rank-ordering ability** in production.

#### **OGM Sample Size Review**

There were no anomalies in the OGM production sample size. Minor increases in sample count were noted between quarters, but changes remained within expected bounds and did not affect the reliability of performance metrics.

---

**Conclusion**:
Based on available OGM data and Wilcoxon Rank-Sum Test results, the model demonstrates **adequate directional performance** during the production period. While formal Gini thresholds cannot be applied due to default scarcity, supporting evidence from Wilcoxon testing and stable predicted PDs affirms the model's continued validity for the 2024 review cycle.

---

Let me know if you'd like a shorter summary version or additional comments on specific buckets or score distributions.
