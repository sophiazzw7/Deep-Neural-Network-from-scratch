Thank you, Phoebe. Based on your request to focus the **overall model performance assessment** on key monitored metrics (AUC, KS, PSI), and to mention supporting metrics (Gini, CAP AUC), here is a revised formal version that avoids listing segment-level values and instead synthesizes findings into a cohesive narrative:

---

## Overall Model Performance Assessment

MRO conducted a comprehensive review of MOD13512’s performance using independent validation testing and ongoing monitoring (OGM) outputs for the 2024 Q2 and Q4 cycles. The assessment confirms that the model continues to demonstrate discriminatory ability across segments but shows degradation in certain areas that warranted escalation. All breaches have been investigated, documented, and addressed.

### Discriminatory Power and Segment Behavior

The model’s ability to separate defaulting from non-defaulting accounts was assessed using AUC, KS, Gini coefficient, and cumulative accuracy profile (CAP) AUC values. These metrics confirm that the model retains segment-level discriminatory power, with consistent rank-ordering across production data. Gini and CAP AUC metrics further support that model predictions align with observed performance and are directionally consistent with expectations.

However, deterioration was observed in the Home Improvement segment, where the AUC breached the firm’s **outer threshold** in 2024 Q2 and the **inner threshold** in Q4. In response, the formal escalation plan—model recalibration or rebuild—was triggered. MRO has verified that a full rebuild is underway, with a vendor-defined delivery timeline. In the interim, a compensating control was implemented, raising the auto-decline score threshold for Home Improvement loans to tighten approval criteria.

Starting in 2024 Q4, KS was formally introduced as a monitored metric. MRO reviewed reported KS values against policy thresholds and confirmed that the **Others segment breached the inner threshold**, reflecting weaker discriminatory power. This breach was documented in the OGM and acknowledged by the developer.

### Population Stability and PSI Assessment

MRO evaluated population stability using PSI for both model features and the model score (ZORS). In Q2, PSI values were largely within acceptable limits, though Home Improvement posted a ZORS PSI above the yellow threshold. This shift coincided with the relaunch of the 20-year loan term, which broadened the applicant pool. MRO confirmed the shift was product-driven rather than evidence of model drift. No recalibration was required.

In Q4, PSI values increased more notably. The Others segment exhibited a red breach in ZORS PSI, while Auto showed a moderate shift. The developer traced these changes to a marketing campaign that generated a spike in low-quality applications. When the PSI was recalculated using only the auto-approved population, all shifts fell within green thresholds, confirming that the model’s core behavior remained stable. MRO validated this explanation and will continue to monitor the impact of such external campaigns on applicant mix.

### Governance and Reporting Enhancements

MRO reviewed the completeness and consistency of OGM reports. The reports are produced quarterly, meeting Tier 2 expectations. However, the Q2 report lacked AUC thresholds and color ratings, which delayed breach recognition and misrepresented performance in SNOW. The developer has since retroactively corrected thresholds and updated the system of record. In addition, PSI thresholds were initially misaligned with OGM standards but will be corrected in future reports.

MRO also confirmed that the pooled “overall” AUC included in earlier OGMs does not reflect a real model and instead represents aggregated segment predictions. Due to known weaknesses in cross-segment comparisons, this metric will be retired going forward.

### Conclusion

Overall, MOD13512 continues to demonstrate adequate performance across most segments, with identified deterioration in Home Improvement actively addressed via model rebuild and interim controls. Population shifts in Q4 were investigated and deemed attributable to business-driven changes rather than model failure. Monitoring metrics—AUC, KS, PSI—are now properly defined, and MRO has confirmed that all significant deviations were escalated and mitigated. The model remains suitable for continued use under the current governance framework.

---

Let me know if you want to attach this as an appendix or blend it into a larger document.
