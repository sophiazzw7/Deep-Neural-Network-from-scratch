Review of Model Assumptions & Limitations

Based on model documentation and analysis provided by the developer, MRO reviewed the following assumptions and limitations to assess their continued reasonableness and validity.

Methodology – The Sheffield PAG model is built on logistic regression hazard models that assume proportionality, linearity, independence of observations, minimal multicollinearity, sufficient sample size, and limited outlier impact. Default and paydown are modeled as competing, absorbing risks, with no allowance for loan cures. MRO considered these methodological assumptions standard and reasonable, as they are consistent with industry practice, and noted that OGM monitoring provides an ongoing check on stability.

Coverage – The model assumes that randomly selected seasoned Sheffield loans are representative of incoming loan pools. The developer acknowledged that variability in collateral content across deals may cause portfolio drift, which cannot be directly measured. MRO considered this assumption reasonable since portfolio drift can be monitored through OGM reporting, although PSI testing may generate false positives.

Data and Input – The development dataset covers a single economic cycle, including the 2008–2010 Great Recession, but excludes post-March 2020 data; therefore, COVID-19 effects are not captured. The dataset begins after 2011, limiting coverage of earlier downturns. Censored loans, such as payoffs or charge-offs, are retained under the assumption that this treatment does not bias model estimates. Key drivers such as origination LTV, term length, product, and manufacturer were unavailable in the development dataset and are excluded. MRO considered these assumptions reasonable given the data constraints, and noted that OGM monitoring during stressed conditions can help assess whether missing factors or limited cycles materially affect performance.

Development – The model simplifies roll and recovery rates by treating them as scalars rather than linking them to macroeconomic conditions. The developer stated this was done to maintain simplicity, and that OGM logs can verify the rates used for deal evaluation. MRO considered this assumption reasonable, as the limitation is transparent and measurable, though continued monitoring is necessary to ensure loss severity remains aligned with observed outcomes.

Implementation – The model is implemented in Excel, and the developer noted risks of manual error, version drift, and dependency on SAS Grid availability. Additionally, correct functioning relies on analysts entering accurate inputs, accessing the refreshed cost of funds file, and being trained to interpret results. MRO considered these assumptions reasonable with compensating process controls, and noted that monitoring log files and OGM outputs helps mitigate operational risks.

Model Use – The model assumes that losses and ROA outputs remain relevant for deal evaluation and that historical data adequately represents future portfolio performance. It also assumes that the exclusion of macroeconomic drivers from the paydown model will not materially bias results, as no intuitive relationship was found between unemployment rates and paydown behavior. MRO considered these assumptions reasonable, given that performance monitoring during downturns will act as a compensating control to highlight any model underperformance due to macroeconomic insensitivity.
* **Implementation in Excel**:
  The developer acknowledges that Excel-based implementation increases the risk of cell errors, version drift, and dependency on SAS Grid availability. This limitation is measurable through process controls, such as log file verification and access monitoring. MRO considered this a valid concern, but also noted that ongoing control checks and end-user training serve as partial mitigants.

---

✅ **Assessment:**
MRO concluded that while the Sheffield PAG model contains several structural and data-related limitations, the developer’s reasoning is generally reasonable. The use of OGM monitoring, log file verification, and performance tracking in downturns provides compensating controls for many of the non-measurable assumptions. However, risks remain around portfolio representativeness, exclusion of macroeconomic drivers, and operational dependency on Excel, which warrant continued monitoring.

---

Would you like me to **tighten the “MRO considered it reasonable…” phrasing** into a standardized one-line conclusion for each bullet (so it looks uniform in an IVR section)?
