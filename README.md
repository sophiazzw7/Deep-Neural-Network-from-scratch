Here is a clear and professional summary of **MRO’s assessment of overall model performance** based on the testing results shown in your images:

---

### **MRO Assessment of Overall Model Performance**

MRO reviewed the performance of MOD 11783 using the results from back testing, ongoing monitoring (OGM), and related supplemental metrics for 2024Q1 through 2024Q4. The assessment focused on the model’s ability to maintain discriminatory power and operational stability under low-default conditions.

#### **Key Findings**:

* **Accuracy Ratio (AR)**: Could not be calculated in any quarter due to fewer than 30 default observations. This is expected given the low-default nature of the portfolio and does not indicate model weakness.

* **Wilcoxon Rank-Sum Test**: Results were satisfactory in all three reviewed quarters (2024Q1, 2024Q3, and 2024Q4), as the default rates in the final four buckets followed the expected increasing order. This indicates the model maintained proper risk rank ordering during the production period.

* **Population Stability Index (PSI)**: PSI results were well below the 25% threshold across all periods, where calculable. Input-level PSI could not be calculated in 2024Q3 due to low obligor counts in each bucket. However, the output-level PSI values supported model stability.

* **External Override Rate**: Only 2024Q3 exceeded the 5% informational threshold (5.83%). The developer investigated and identified the cause as system or timing-related events—not true model overrides. The issue was addressed by increasing OGM frequency to biannual. The rate returned to a satisfactory level (0.14%) in 2024Q4.

* **Back Testing**: Gini coefficients for the PDB model were calculated as early readings for three quarters (0.62 in Q2, 0.50 in Q3, 0.47 in Q4). PDA Gini could not be reliably calculated due to insufficient defaults. Although results are directional only, they provide evidence that the model maintained discriminatory power. These findings are further supported by consistent results from the Wilcoxon Rank-Sum Test.

---

### **Conclusion**

Based on available data, MRO concludes that MOD 11783 is **performing as intended**. Despite the limitations imposed by low default counts, the model continues to demonstrate adequate discriminatory power and portfolio stability. All identified exceptions were either expected (e.g., AR not calculated) or fully investigated and mitigated (e.g., Q3 override rate). Therefore, **no concerns or risk items are raised**, and the model remains suitable for continued use in production.







