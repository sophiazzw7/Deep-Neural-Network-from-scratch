Thank you for the clarification. Here's the revised version of the section, now accurately reflecting that a breach **did occur** in 2024Q3, but was **appropriately investigated and addressed**, without stating "no breach occurred":

---

### 9.6 Model Overlays

MOD 11783 operates within the PRISM Batch Process and does not allow users to manually override the model-generated ratings within the model itself. Any changes to risk ratings occur outside the batch model—typically initiated by the Credit Lending Unit (CLU) as part of the business process, such as assigning a Criticized or Classified rating. These changes, referred to as **external-model overrides**, are inherently different from overrides in models that involve direct user interaction.

Previously, external overrides were treated as a performance metric in the Ongoing Monitoring (OGM) framework. However, beginning in 2025Q1, this metric has been reclassified as informational only, and this change was approved by MRO during the most recent model validation.

In 2024Q3, the external override rate was reported at **5.83%**, breaching the applicable threshold. The model developer investigated the cause of the breach in collaboration with the model user. The analysis determined that the elevated override rate was primarily driven by operational scenarios such as batch timing delays, pass-through ratings from related systems, and obligors crossing the \$2MM threshold triggering a transition to a non-batch model. Only one case was identified that might arguably be considered a discretionary override, and even that classification was not clear-cut. The majority of rerated cases reflected **automated or structural model logic**, rather than manual intervention.

To mitigate the issue, the developer committed to enhanced monitoring, increasing the OGM reporting cadence from the Tier 3 requirement of annual reporting to a **bi-annual schedule**. In practice, reports were already produced for 2024Q1 and 2024Q2, and an additional report was generated for 2024Q4. Following these efforts, the external override rate in 2024Q4 fell significantly to **0.14%**.

\| **Table 13: External Override Rate** |
\|--------------------------|-------------------------|-----------------------------|
\| Quarter End Month        | Total Obligor Count     | Obligors Rerated Count      |
\| **202403**               | 2,239                   | 102                         |
\| **202409**               | 2,076                   | 121                         |
\| **202412**               | 2,106                   | 3                           |

**MRO Assessment:** The override threshold breach in 2024Q3 was investigated and appropriately addressed. Given the system-driven nature of the rerated cases, the clarification provided by the model user, and the strengthened monitoring controls, **MRO considers the issue resolved and no risk item is raised**.

---

Let me know if you want to include a reference to section 9.4.2.1 or note the removal of the metric from performance tracking more explicitly.







