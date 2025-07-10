### Exhibit A

**Assessment of the “Overall” AUC that combines all four loan-purpose models**

MRO disassembled the pooled AUC calculation that appears in the 2024 Q2 and Q4 OGM reports and compared it with the four individual segment-level AUCs.  The investigation produced three key observations:

1. **Cross-segment pairs dominate the statistic.**
   When every positive-negative application pair used in the pooled AUC is labelled by segment, approximately **71 percent** of the pairs span two different loan-purpose segments.  The combined AUC therefore measures how the four segment score scales line up against one another, not how well a single model ranks borrowers within a common population.

2. **Cross-segment comparisons are inherently unstable.**
   In the 2024 Q2 data, for example, the average score for a non-defaulted borrower in the Debt-Consolidation segment (-0.047) is higher than the average score for a defaulted borrower in the Auto segment (-0.039).  This scale mismatch can cause the pooled AUC to treat “apples-to-oranges” comparisons as evidence of good rank ordering even when no segment model has changed.

3. **A small change in segment mix can move the metric sharply.**
   A minor volume shift, or a calibration update in just one segment, can raise or lower the pooled AUC by several points without indicating genuine model deterioration.

On the basis of this analysis MRO concludes that the pooled “overall” AUC does not provide a meaningful performance signal and recommends dropping that metric from future OGM reports.  The developer has agreed, and the “overall” line item will be suppressed beginning with the 2025 Q1 cycle.

---

### Amended wording for the Q2 2024 PSI narrative

The Q2 PSI review showed that Home Improvement recorded a ZORS PSI of **0.13**, exceeding the 0.10 warning threshold and placing the shift in the “moderate” range.  According to the developer, this movement coincided with the **re-launch of the twenty-year Home-Improvement loan term in mid-April 2024**, which broadened the applicant pool to include borrowers requesting larger amounts and longer maturities.  MRO confirmed the explanation by:

* comparing pre- and post-April score distributions, which revealed a noticeable influx of higher-balance applications, and
* recalculating PSI on the legacy fifteen-year term alone, which reduced the ZORS PSI to **0.06**, comfortably within the green zone.

Because the shift was attributable to a controlled product change rather than model drift, MRO agreed that **no immediate recalibration was necessary**.  The developer documented the product launch, and MRO requested that future OGMs continue to track the twenty-year sub-population separately for at least two additional quarters to ensure that the distribution stabilises.

---
