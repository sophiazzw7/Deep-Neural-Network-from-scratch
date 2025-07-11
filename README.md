Certainly! Here's a polished and professional version of the paragraph you're looking for, with proper context inserted for both the AUC and KS threshold changes:

---

This review includes a change-based assessment of the OGM monitoring plan. Two key updates were made by the model developer in response to prior feedback and performance observations:

1. **Removal of AUC Threshold for Overall (Pooled) Segment**
   The AUC threshold for the overall (pooled) population was removed from both Appendix G and the OGM performance monitoring section. This change reflects MRO’s prior recommendation that overall metrics—aggregated across loan-purpose segments—can obscure segment-specific degradation and should not be used to drive model escalation decisions. Instead, performance thresholds should be applied at the segment level to ensure early and targeted detection of model deterioration.

2. **Update to KS Threshold Methodology**
   The KS threshold methodology has been revised to address previous implementation issues and to improve monitoring stability. The developer corrected the use of the KS statistic to align with the two-sample KS definition documented in the development file, and recalculated thresholds using pooled development data rather than volatile quarterly estimates. The revised thresholds are now based on a bootstrap-derived mean and standard deviation from the pooled dataset, eliminating the prior “minimum-of-five” rule, which had caused thresholds to be overly conservative due to outlier variance in individual quarters. This change ensures thresholds are both statistically sound and operationally usable.

---

Let me know if you want this adapted for use in a slide, appendix note, or executive summary.
