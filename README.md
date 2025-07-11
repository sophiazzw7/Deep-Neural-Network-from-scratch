Certainly, Phoebe. Below is your revised draft with the two requested changes:

---

**Assessment of KS Threshold Methodology**

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

While the overall bootstrap framework was methodologically sound, its implementation contained a critical error: the developer had inadvertently used the wrong KS formula in the SAS code, which was inconsistent with the two-sample KS statistic (PROC NPAR1WAY, statistic D) specified in the development documentation. As a result, the calculated thresholds were unrealistically low. For example, for the Auto segment, the red KS threshold was calculated as 0.0024 and the yellow threshold as 0.0043. Similar thresholds for other segments (e.g., Home Improvement red threshold of 0.0299; Others red threshold of 0.0044) were far below historical norms and would have triggered excessive false alerts. These issues stemmed not from the bootstrap methodology itself, but from the misimplementation of the KS metric.

After MRO raised this issue, the developer acknowledged the error, corrected the KS calculation, and regenerated the thresholds accordingly. The updated derivation used the proper two-sample KS metric and a consistent bootstrap procedure.

MRO reviewed the corrected thresholds and proposed two final refinements to improve monitoring stability. The existing method for setting thresholds was:

1. Drawing 1,000 bootstrap samples per quarter (Q1–Q4 plus “All Quarters”).
2. Computing mean (μ) and standard deviation (σ) for each quarter.
3. Defining thresholds: Red = μ – 4σ; Green = μ – 3σ.
4. Selecting the minimum of the five quarterly thresholds.

This approach performed well for AUC, which has relatively low variability (σ \~0.02). However, KS values displayed much higher volatility (σ ranging from \~0.035 to 0.09), which made the minimum-of-five approach overly conservative in some cases. In particular, the Q2 bootstrap results had inflated standard deviations and substantially lowered the thresholds across several segments:

| Segment            | Q4 Observed KS | Current Red Threshold | Decline Required to Breach |
| ------------------ | -------------- | --------------------- | -------------------------- |
| Auto               | 0.380          | 0.1616 (set by Q2)    | \~57%                      |
| Debt Consolidation | 0.277          | 0.1310 (set by Q2)    | \~59%                      |
| Home Improvement   | 0.282          | 0.2475                | \~44%                      |
| Others             | 0.270          | 0.1540 (set by Q2)    | \~60%                      |

For example, the Auto segment had a Q2 KS standard deviation of 0.0773—more than double the pooled standard deviation of 0.0350—thus skewing the final thresholds downward and allowing substantial performance deterioration before triggering alerts.

To address these issues, MRO proposed the following refinements:

1. **Use pooled development data across all quarters ("All Quarters")** to mitigate the undue influence of any single volatile quarter.
2. **Eliminate the “minimum-of-five” threshold rule**, which disproportionately amplified the impact of outlier standard deviations and introduced instability.

Under the revised and final approach, the KS thresholds were recalculated using the pooled development data. The final thresholds for each segment were:

| Segment            | Mean (μ) | Std Dev (σ) | Red (μ − 4σ) | Yellow (μ − 3σ) |
| ------------------ | -------- | ----------- | ------------ | --------------- |
| Auto               | 0.3970   | 0.0350      | 0.2570       | 0.2920          |
| Debt Consolidation | 0.3178   | 0.0155      | 0.2558       | 0.2713          |
| Home Improvement   | 0.4412   | 0.0273      | 0.3320       | 0.3593          |
| Others             | 0.3841   | 0.0288      | 0.2689       | 0.2977          |

This final configuration aligns well with both the development and recent production KS statistics. For example, the Q4 observed KS values (Auto: 0.380, Debt Consolidation: 0.277, Home Improvement: 0.282, Others: 0.270) all fall within the new control bands—confirming that current performance is near expectations while still allowing early detection of future deterioration.

**Conclusion:**
MRO concludes that the final KS threshold methodology—grounded in pooled development baselines, a corrected statistical implementation, and removal of overconservative selection rules—is conceptually sound and operationally effective. It achieves the dual objectives of stability and sensitivity, supporting robust ongoing model performance monitoring.

---

Let me know if you'd like this broken into a formal memo format or shortened for a presentation slide.
