Here’s a revised draft that weaves in the **baseline explanation** alongside the **thresholds** and **assessment narrative** in a natural, professional way:

---

### Stability Testing Assessment

Stability testing was conducted using two early warning indicators: the directional shift metric and the population stability index (PSI). Together, these measures assess whether the model portfolio remains stable over time or shows signs of systematic migration.

For the directional shift, the baseline portfolio is defined as the set of obligors and obligations rated 12 months prior to the current OGM quarter. This baseline requirement ensures that results reflect rating migrations over a full year, rather than short-term fluctuations. Ratings from the baseline period are compared to the current quarter for the same obligors, using the most recent rating available within the 12-month window.

The directional shift threshold is set so that values between -1.0 and 1.0 indicate balanced rating migrations, while values outside this range signal material rating shifts. Across the 2024Q2 to 2025Q1 review period, the directional shift results fell within the acceptable range, with test values of -0.08 in 2024Q2, -0.06 in 2024Q3, -0.09 in 2024Q4, and -0.34 in 2025Q1. These results show that the portfolio remained broadly stable, with no evidence of material migration.

For PSI testing, the established threshold is 0.25. Results less than or equal to this threshold indicate an acceptable population shift, while results greater than 0.25 suggest a significant population shift requiring attention. In practice, many PSI metrics returned “NA” assessments because of insufficient counts in key input buckets, particularly for Non-ABL exposures. Where results were available, they were within the acceptable range. For example, Non-ABL control type in 2025Q1 showed a PSI of 0.038, which indicates an acceptable population shift.

Overall, the stability testing results indicate that rating migrations and key input distributions have remained within acceptable levels. While the limited sample sizes restricted PSI testing in several quarters, the directional shift metric—anchored to the baseline portfolio—provides assurance that the model portfolio has not experienced abnormal migration trends.

---

Would you like me to also contrast this with the **developer’s OGM stability reporting** (the Excel outputs you shared), so that your section makes clear where MRO’s independent interpretation adds value?
