Got it — I’ll restructure the content into **bullets with full sentences**, keeping the developer’s stated logic intact and layering on assessment (MRO view) where monitoring or “not measurable” is noted. This way you can directly use it in your validation.

---

# **Assumptions and Limitations**

## **Key Assumptions**

* The model development relies on a single economic cycle, which includes the Great Recession of 2008–2010 but excludes post-March 2020 data; therefore, impacts of the COVID-19 pandemic are not captured.
* Historical data are assumed to adequately represent future portfolio performance, and randomly selected seasoned Sheffield loans are assumed to be representative of incoming loan pools.
* Censored loans, such as early payoffs or charge-offs, are retained in the estimation dataset under the assumption that they do not bias model results.
* Default and paydown are modeled as absorbing and competing risks, which means that once a loan defaults or pays down, no cure is modeled.
* The model adheres to standard logistic regression hazard assumptions: binary dependent variable, linearity of predictors with log-odds, independence of observations, limited multicollinearity, sufficient sample size, and limited outlier impact.
* Balance weighting is used to preserve aggregate portfolio accuracy, although this may reduce accuracy at the loan level.
* Implementation assumes that analysts provide accurate deal inputs, that the cost of funds file is refreshed daily and accessible, and that analysts are properly trained to interpret results.

---

## **Key Limitations and Monitoring**

* **Representativeness of development data**:
  The developer assumes that seasoned Sheffield loans are representative of incoming loans. This assumption is not directly measurable, and the developer notes that variability in collateral content across PAG deals makes comparison difficult. While PSI analysis can be attempted, it may generate false positives. MRO considered the developer’s reasoning reasonable, as ongoing OGM monitoring can highlight portfolio drift even if not fully captured by PSI tests.

* **Roll and recovery rate simplification**:
  Roll and recovery rates are modeled as scalars rather than linked to macroeconomic drivers, with the stated reason being to maintain simplicity. This assumption is measurable, as OGM can compare roll and recovery rates used in deal evaluations against development-period benchmarks. MRO found this approach reasonable, since OGM log files can verify that loss severity assumptions remain consistent.

* **Forecasts tied to economic environment**:
  The developer acknowledges that forecasts depend on economic conditions and that future recessions with characteristics different from the Great Recession could cause greater variance between observed and estimated losses. This assumption is not directly measurable, but the developer indicates that OGM monitoring based on historical macroeconomic data can mitigate these risks. MRO considered this explanation reasonable because monitoring during downturns provides a compensating control.

* **Exclusion of key drivers**:
  Origination LTV, term length, product, and manufacturer are not consistently available across the development dataset, and their exclusion may cause payment default or paydown risk to be misstated during recessionary periods. This assumption is not measurable, but the developer states that performance monitoring during downturns can highlight the impact of missing drivers. MRO considered this reasonable, given that monitoring provides an indirect check until additional drivers become available.

* **Macroeconomic insensitivity of paydown model**:
  The developer tested unemployment rates and unemployment claims but found no intuitive relationship to paydown; as a result, no macroeconomic variables were included. This assumption is not measurable, but the developer notes that downturn monitoring can surface this limitation. MRO considered the reasoning acceptable, as future OGM cycles during stressed conditions will allow testing of whether exclusion of macro drivers results in material deviations.

* **Data coverage limited to post-2011**:
  The dataset begins after 2011, and the developer notes that later recessions (e.g., COVID-19) will be tested in future iterations. This assumption is not measurable in the current framework. MRO considered the reasoning reasonable, as the limitation is acknowledged and deferred to future model updates.

* **Aggregate weighting over loan-level accuracy**:
  The model uses balance weighting to ensure accuracy at the aggregate level, but the developer acknowledges this may mask loan-level shortfalls. This limitation is measurable through OGM loan-level backtesting. MRO considered this a reasonable design trade-off, as aggregate portfolio accuracy is the model’s stated purpose, with monitoring at finer granularity serving as a compensating measure.

* **Implementation in Excel**:
  The developer acknowledges that Excel-based implementation increases the risk of cell errors, version drift, and dependency on SAS Grid availability. This limitation is measurable through process controls, such as log file verification and access monitoring. MRO considered this a valid concern, but also noted that ongoing control checks and end-user training serve as partial mitigants.

---

✅ **Assessment:**
MRO concluded that while the Sheffield PAG model contains several structural and data-related limitations, the developer’s reasoning is generally reasonable. The use of OGM monitoring, log file verification, and performance tracking in downturns provides compensating controls for many of the non-measurable assumptions. However, risks remain around portfolio representativeness, exclusion of macroeconomic drivers, and operational dependency on Excel, which warrant continued monitoring.

---

Would you like me to **tighten the “MRO considered it reasonable…” phrasing** into a standardized one-line conclusion for each bullet (so it looks uniform in an IVR section)?
