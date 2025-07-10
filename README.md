The model’s on-going governance memorandum (OGM) is produced quarterly, satisfying the Tier 2 frequency requirement. The OGM tracked two metrics—AUC and PSI—in 2024 Q2, and three metrics—AUC, PSI and KS—in 2024 Q4. MRO has confirmed that all monitoring results have been reported, reviewed and archived in the model-risk repository for both quarters.

5.1 Completeness and documentation review
During its review MRO identified several documentation gaps and has coordinated corrective actions with the developer:

AUC results lacked embedded thresholds and colour ratings. The omission delayed the formal trigger of an action plan when the Home-Improvement AUC breached the outer threshold in Q2, and it caused an incorrect “Green” performance status to be posted in ServiceNow (SNOW). The developer has added threshold annotations to the AUC tables and has corrected the SNOW rating retroactively.

PSI tables continued to use Early-Warning Indicator (EWI) thresholds (0.25 / 0.40) rather than the current OGM standards (0.10 / 0.25). The developer has committed to apply the proper OGM thresholds in all future reports.

Rank-order tables and an OGM action-plan tracker were missing. MRO has requested that both items be incorporated to provide clear evidence of segment-level discriminatory power and to document the status of any outstanding remediation work.

The OGM included an “overall” model segment that combines predictions across the four individual segment models. MRO verified with the developer that no single pooled model exists; the pooled AUC therefore measures cross-segment scale differences rather than genuine model performance. MRO’s independent analysis (see Exhibit A) confirmed the conceptual weakness of this metric, and the overall segment will be removed from future OGMs.

5.2 Threshold breaches and investigative follow-up
AUC breaches. Home Improvement breached the firm’s outer AUC threshold in Q2 2024 and the inner (yellow) threshold in Q4 2024. The escalation requirement is recalibration or rebuild. MRO has verified that the developer has triggered a full rebuild; a rebuild timeline has been provided, and work is under way. While the rebuild proceeds, the model user has imposed a compensating control, raising the Home-Improvement auto-decline cut-off from 630 to 660.

KS breach. KS was introduced as a monitored metric in Q4 2024. The Others segment breached the inner KS threshold. MRO confirmed that the developer has opened an investigation; no scorecard degradation has been found and the metric will be tracked closely in subsequent quarters.

PSI analysis.

Q2 2024. All feature PSI values were below 0.11. Home Improvement posted 0.101, technically a “Yellow” condition under policy. ZORS PSI for Home Improvement was 0.13, reflecting the re-launch of the twenty-year term; the moderate shift was documented and no recalibration was warranted.

Q4 2024. The Others segment exhibited a ZORS PSI of 0.395—a “Red” breach—and Auto showed a moderate ZORS shift of 0.127. Feature PSI for Others reached 0.14. The developer traced these shifts to a November 2024 digital-marketing campaign that generated a surge of low-quality applications. When PSI was recalculated on the auto-approved population only, all values fell below 0.10 (e.g., ZORS PSI for Others dropped to 0.005). MRO accepts this explanation and will ensure that campaign effects are isolated in future monitoring cycles.

All deviations have therefore been investigated, documented and either remediated or placed under active follow-up, and the associated action items are tracked in the updated OGM action-plan table.

5.3 Conclusion
Taking into account the corrected documentation, the ongoing rebuild of the Home-Improvement scorecard, the interim compensating control, and the root-cause analysis of the Q4 PSI movements, MRO is satisfied that the model’s performance monitoring framework now meets policy expectations. The governance team will continue to verify, on a quarterly basis, that threshold annotations remain in place, that the new KS and PSI standards are applied, and that progress against the rebuild timeline is recorded.
