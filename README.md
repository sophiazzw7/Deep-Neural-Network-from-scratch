Here’s a polished version of your draft incorporating both your analysis and the content from the image:

---

### Assessment of Compensating Measures

During the 2025 annual review of MOD13512, MRO noted multiple signals of model performance deterioration in the Home Improvement segment. Specifically, the 2024Q2 OGM report showed an **outer breach in AUC**, and the 2024Q4 OGM report indicated an **inner breach in AUC** for the same segment. In parallel, **KS statistics also evidenced degradation**, with an **outer breach observed in the 2024Q4 KS** for Home Improvement. Based on these findings, a model rebuild has been initiated.

Given the rebuild timeline, the model developer proposed a set of **compensating measures** as an interim control. These measures, detailed in the updated OGM documentation, aim to mitigate risk exposure until the new model is deployed. For the Home Improvement segment, the proposed actions include:

* Increasing the **auto-decline FICO cutoff from 630 to 660** (implemented on 7/23).
* Implementing **auto-decline for credit history <5 years** (timeline TBD, likely by Sept/Oct).
* Implementing **auto-decline for credit builders with Open Secured Card** (timeline TBD, likely by Sept/Oct).
* Implementing **auto-decline for current delinquents with Open Secured Card** (timeline TBD, likely by Sept/Oct).

These changes are designed to tighten underwriting standards for higher-risk applicants based on non-model criteria, as the model alone no longer reliably separates good and bad loans in this segment.

Since the compensating measures were introduced post hoc and outside of a controlled testing framework, MRO cannot directly compare model performance with and without the measures at this time. However, MRO will evaluate their impact in future reviews by tracking trends in observed credit performance, approval mix, and model metrics. This evaluation will help determine whether the interim measures were effective in mitigating the observed degradation.

MRO supports the inclusion of these compensating controls as a reasonable short-term response, but reiterates that the long-term solution remains the successful deployment of a recalibrated model.

---

Let me know if you'd like a version formatted for inclusion in a report or slide deck.
