Certainly — here's the revised version focusing primarily on **Gini metrics** from the OGM reports, with the **Wilcoxon Rank-Sum Test** included as supplemental evidence. The language strictly reflects only the content you've provided:

---

### **9.2 Back Testing — Production Period**

During the production period, model performance was assessed through Ongoing Monitoring (OGM) reports, with a focus on evaluating the discriminatory power of the model using Gini coefficients. As a supplemental approach, the Wilcoxon Rank-Sum Test was also applied due to limitations in observed defaults.

#### **OGM Gini Coefficient Review**

Gini coefficients were calculated for PDA and PDB across three recent quarters: 202306, 202309, and 202312. The number of observed defaults was limited in each period, particularly for PDA:

* **PDA Default Counts**:

  * 202306: 4
  * 202309: 6
  * 202312: 8

* **PDB Default Counts**:

  * 202306: 12
  * 202309: 21
  * 202312: 26

Given that fewer than 30 defaults were observed in each period, the results are considered **early readings** and do not support formal threshold-based conclusions per model policy. However, Gini coefficients were as follows for PDA:

* **Gini (PDA)**:

  * 202306: 0.62
  * 202309: 0.50
  * 202312: 0.47

These values suggest directional evidence of discriminatory ability, though limited by low default volume.

#### **Wilcoxon Rank-Sum Test (Supplemental)**

To supplement the Gini analysis, the Wilcoxon Rank-Sum Test was applied to multiple model score variants (`Cal_RR`, `Kappa_RR`, and `Final_RR`) for both PDA and PDB. In all cases, the default rates in the final four buckets followed the expected increasing order, indicating the model **passed the Wilcoxon Rank-Sum Test**. For example, in the PDB `Final_RR`, default counts in Classes 2–5 were 16, 52, 67, and 52, respectively.

#### **OGM Sample Size Review**

The OGM data sample size increased slightly compared to the prior quarter. The documentation notes:

> “There is no anomaly in the OGM data sample size.”

---

**Conclusion:**
Based on Gini values reported in OGM and supplemental results from the Wilcoxon Rank-Sum Test, the model demonstrates evidence of discriminatory power during the production period. Due to limited default counts, the findings are considered directional rather than conclusive but support continued model use in production.

---

Let me know if you'd like to integrate commentary on calibration metrics or exclude Wilcoxon details further.
