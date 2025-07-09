Certainly, Phoebe. Here's a revised, formal version of the PSI analysis write-up that separates ZORS and feature PSI by quarter, incorporates the correct threshold terminology ("standard OGM PSI threshold" vs. "EWI OGM PSI threshold"), and provides a clear MRO independent assessment where appropriate:

---

### PSI Analysis Summary – 2024 Q2 and Q4

To assess model stability, both ZORS and feature distributions were evaluated using the Population Stability Index (PSI) for 2024 Q2 and Q4. The standard OGM PSI threshold defines values below 0.10 as “no significant shift,” between 0.10 and 0.25 as a “moderate shift,” and values above 0.25 as a “significant shift.” In contrast, the developer used the EWI OGM PSI threshold, which has a higher tolerance range (e.g., up to 0.11 for “no shift” and 0.27 for “significant shift”).

---

#### **2024 Q2 – ZORS PSI**

According to the developer’s analysis, all ZORS PSI values in 2024 Q2 fall below the EWI OGM threshold. However, when applying the standard OGM PSI threshold, the Home Improvement segment showed a PSI of 0.13, indicating a moderate shift. The developer attributed this to the reactivation of the 20-year Home Improvement loan option in April 2024, which, according to their view, introduced a different applicant population with distinct credit profiles.

MRO independently reviewed this explanation. Given the timing of the reactivation and the segment affected, the observed moderate shift appears reasonable and consistent with expectations. The change is isolated to the Home Improvement segment, while other segments remained stable. Based on available OGM data, MRO finds the developer’s explanation plausible, though we note that the dataset does not include granular indicators to directly confirm the shift in loan terms. Nonetheless, the ZORS PSI result is consistent with a population change driven by a product reintroduction.

---

#### **2024 Q2 – Feature PSI**

Feature PSI for all segments in Q2 remained below the EWI OGM threshold. Under the standard OGM PSI threshold, all segments also fall below the 0.10 threshold, except for Home Improvement (0.101), which sits just above it. While the developer still considered this Green using EWI thresholds, under standard OGM thresholds this would be classified as a borderline moderate shift.

No clear business event was cited by the developer for this feature drift. MRO’s assessment is that the small elevation is likely related to the same product change affecting ZORS. While the PSI is near the threshold, it does not raise immediate concern, but should be monitored in subsequent quarters.

---

#### **2024 Q4 – ZORS PSI**

In 2024 Q4, ZORS PSI values showed more noticeable shifts. According to the developer’s report, the “Others” segment exhibited a PSI of 0.395, breaching both the standard and EWI OGM thresholds and representing a significant shift. The Auto segment also showed a PSI of 0.127, which falls into the moderate shift category under the standard threshold.

The developer attributed these shifts to a digital marketing campaign launched in November 2024, including Google Ads and paid social media. This campaign drove a surge in applications, most of which were auto-declined due to poor credit quality. The developer reported that in Q4, 279,773 applications were received, with 203,389 auto-declined. For comparison, Q2 had 123,118 total applications, of which 83,248 were declined.

MRO independently verified these application volumes using the same dataset and found matching figures: 279,773 total applications in Q4 with 203,389 declined, and 123,118 total applications in Q2 with 83,248 declined. While the OGM dataset lacks detailed campaign-level flags to confirm the marketing source, the volume shift aligns with the developer’s explanation. MRO finds it reasonable to conclude that the observed ZORS PSI shifts are driven by a temporary change in applicant mix caused by the campaign, not model degradation.

To further isolate model performance, the developer re-ran PSI using only the auto-approved population. Under this approach, all ZORS PSI values for Q4 returned to below 0.10, confirming no underlying shift in the scored population used for decisioning. MRO agrees that this filtered analysis provides a more accurate view of model stability.

---

#### **2024 Q4 – Feature PSI**

Feature PSI values in Q4 also show increased divergence. Using the full population, the “Others” segment reached 0.14, which is a moderate shift under the standard OGM threshold. The developer again reran the analysis using only auto-approved loans. Under this filtered view, all PSI values except Debt Consolidation (0.116) dropped back below the 0.10 threshold.

The developer provided a deeper dive on the Debt Consolidation model, identifying that the highest PSI-contributing feature was the average credit limit on active revolving accounts (`dIAmt_active_accts__mean_by_prtfType_revolving`). They showed that credit limits for approved applicants in production were significantly higher than in the development sample. MRO reviewed this explanation and the accompanying histogram and agrees that the feature shift reflects improved applicant quality rather than model instability.

---

### Summary

Across both quarters, the developer applied the EWI OGM PSI threshold, while MRO evaluated results against the stricter standard OGM PSI threshold. In 2024 Q2, only minor shifts were observed. In 2024 Q4, significant shifts appeared in the unfiltered population, particularly in the “Others” segment. However, after filtering to auto-approved applicants, all PSI values were within expected limits.

MRO independently validated the developer’s population counts and agrees that the shifts were primarily caused by temporary acquisition events rather than model drift. No model change is recommended at this time, though continued monitoring is advised—particularly in segments with elevated feature drift (e.g., Debt Consolidation).

Let me know if you'd like this version formatted into PowerPoint or Word.
