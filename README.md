Certainly! Here's the revised version of the KS threshold assessment write-up, now integrating full context from the photos without referring to them explicitly:

---

### Assessment of KS Threshold Methodology

As part of the 2025 annual review of MOD13512, MRO conducted a comprehensive reassessment of the methodology used to define performance thresholds for the Kolmogorov–Smirnov (KS) statistic. Over the past year, several iterations of the KS threshold methodology were developed and updated following feedback from model governance and independent review groups.

Initially, the model developer proposed deriving the KS thresholds based on hypothesis testing using a p-value of 0.05. However, the governance team advised that performance thresholds should be based on model expectations established during development, rather than on significance testing, which is sensitive to sample size and not intended for monitoring purposes.

In response, the developer updated the KS thresholds in MDD version 20250515 using the 2024Q4 production (OGM) data as the baseline. Under this version, inner and outer breach levels were defined as ±5% and ±10% deviations from the Q4 KS values. MRO raised concerns with this approach, noting that MOD13512 had already shown evidence of discrimination degradation as early as the 2024Q2 monitoring, and therefore using Q4 metrics as a baseline would embed the deterioration into the control logic and allow further degradation to go undetected.

MRO communicated that the KS threshold should instead be anchored to the development data. In response, the developer proposed a bootstrap-based thresholding framework. Specifically:

* 1,000 bootstrap samples were drawn with replacement from the development dataset for each loan-purpose segment.
* KS was calculated for each bootstrap sample between the model score and target.
* The mean (μ) and standard deviation (σ) of the 1,000 KS values were computed.
* Thresholds were defined as follows:
  • Red: KS ≤ μ − 4σ
  • Yellow: μ − 4σ < KS ≤ μ − 3σ
  • Green: KS > μ − 3σ

To stabilize the calculation and reduce volatility, thresholds were computed both quarterly and across the entire development sample. The final lower (red) and upper (green) thresholds for each segment were determined by taking the minimum value among the five lower (μ − 4σ) and five upper (μ − 3σ) thresholds, respectively, with the constraint that the KS threshold must remain positive.

This method, however, resulted in thresholds that were orders of magnitude too low. For example, for the Auto segment, the red KS threshold was calculated as 0.0024 and the yellow threshold as 0.0043. Similar threshold levels were produced for the other segments (e.g., Home Improvement red threshold of 0.0299; Others red threshold of 0.0044), which were inconsistent with historical KS values and unreasonably tight. These overly conservative limits stemmed from a technical issue: the developer had inadvertently used an incorrect KS definition not aligned with the two-sample KS statistic documented in the model development file (PROC NPAR1WAY, statistic D).

After MRO raised this issue, the developer acknowledged the error, corrected the KS calculation, and regenerated the thresholds accordingly. The updated derivation used the proper two-sample KS metric and a consistent bootstrap procedure.

MRO reviewed the corrected thresholds and proposed two final refinements to improve monitoring stability:

1. **Use pooled development data across all quarters** to mitigate threshold sensitivity to seasonal volatility.
2. **Eliminate the “minimum-of-five” rule**, which had been overly conservative in some cases due to anomalous standard deviations (e.g., Q2 standard deviation for Auto segment = 0.0773 vs. pooled σ = 0.0350).

Under the revised and final approach, the KS thresholds were recalculated using the pooled "All Quarters" development data. The final thresholds for each segment were:

| Segment            | Mean (μ) | Std Dev (σ) | Red (μ − 4σ) | Yellow (μ − 3σ) |
| ------------------ | -------- | ----------- | ------------ | --------------- |
| Auto               | 0.3970   | 0.0350      | 0.2570       | 0.2920          |
| Debt Consolidation | 0.3178   | 0.0155      | 0.2558       | 0.2713          |
| Home Improvement   | 0.4412   | 0.0273      | 0.3320       | 0.3593          |
| Others             | 0.3841   | 0.0288      | 0.2689       | 0.2977          |

This final configuration aligned well with the observed KS values from both development and production datasets. For example, the Q4 observed KS values (Auto: 0.380, Debt Consolidation: 0.277, Home Improvement: 0.282, Others: 0.270) fall within the new control bands and demonstrate that current performance is near expected levels while still allowing room for early detection of future deterioration.

MRO concludes that the final methodology—based on pooled development baselines, correctly specified KS metrics, and statistically principled control limits—is sound. It appropriately balances responsiveness to model degradation with stability, and avoids the pitfalls of relying on volatile production metrics or overly narrow control bands.

---

Let me know if you’d like this section formatted for a presentation or audit appendix.
