Perfect. I’ll now expand **all major assumptions and limitations** with the same level of detail as I gave you for PSI/coverage, ensuring each bullet includes:

1. Developer’s reasoning,
2. Measurability (if stated),
3. Monitoring/compensating controls,
4. MRO’s assessment.

---

# **Review of Model Assumptions & Limitations**

* **Coverage –** The model assumes that randomly selected seasoned Sheffield loans are representative of incoming loan pools. The developer acknowledged that collateral content varies significantly across PAG deals, making it difficult to directly compare the development sample to the current portfolio. While PSI testing can be used to evaluate portfolio drift, the developer noted that PSI results may generate false positives, since temporary mix changes by product or manufacturer may not signal true model risk. MRO considered this assumption reasonable because portfolio representativeness can be monitored through OGM reports that evaluate default and paydown performance directly. PSI analysis can provide a directional check, but OGM monitoring offers the stronger control.

* **Roll and Recovery Rates –** The model simplifies roll and recovery rates by using scalar values rather than linking them to macroeconomic drivers. The developer stated this was done to maintain simplicity and transparency, and noted that OGM log files can verify the specific roll and recovery rates applied in deal evaluations. This assumption is measurable since OGM can confirm whether loss severity assumptions remain consistent with expectations. MRO considered this reasonable, while emphasizing that ongoing monitoring is necessary to ensure scalar assumptions remain aligned with observed outcomes, particularly in stressed environments.

* **Exclusion of Macroeconomic Drivers in Paydown –** The paydown model does not include macroeconomic factors such as unemployment rates or claims, because the developer found no intuitive or stable relationship between these variables and paydown performance. As a result, the paydown model is not sensitive to changes in the economy. The developer acknowledged this as a limitation, noting that future monitoring during downturns (e.g., COVID-19) would allow for reassessment. This assumption is not directly measurable at present, but OGM monitoring during stressed environments provides a compensating control. MRO considered this reasonable, given the lack of clear empirical relationships, but highlighted the need for continued monitoring.

* **Data Coverage –** The development dataset begins after 2011, includes the Great Recession period, and excludes all data after March 2020. This means COVID-19 impacts and later recessions are not captured in the current version. The developer acknowledged this limitation and committed to testing future cycles once data are available. This assumption is not directly measurable today, but OGM monitoring and future redevelopment will provide coverage. MRO considered this limitation reasonable given the timing of available data, but noted that testing with later cycles should be prioritized.

* **Censoring –** The developer recognizes that loans may exit the dataset prior to payoff or charge-off, which creates potential censoring. The treatment applied allows censored loans to remain in estimation under the assumption that they do not bias model results. This assumption is supported by reference to RQCRA methodology. It is not directly measurable, but OGM monitoring provides an indirect check by comparing censored versus completed loans. MRO considered this approach reasonable, since censoring is acknowledged, and compensating analysis exists.

* **Aggregate Weighting –** The model uses balance weighting to ensure accuracy at the portfolio level, but the developer acknowledges this may mask mis-estimation at the loan level. This assumption is measurable through OGM testing, which reports discrimination and calibration at the loan level. MRO considered this a reasonable trade-off, as the model is intended for aggregate forecasting, and loan-level monitoring can supplement portfolio accuracy.

* **Implementation Risks –** The model is implemented in Excel, which introduces risks of cell-level calculation errors, version drift, and dependency on SAS Grid availability. The developer acknowledged these risks and noted that monitoring and user training provide compensating controls. This limitation is measurable through process checks, such as validation of log files and version control reviews. MRO considered these risks reasonable and manageable, given the existing compensating measures.

* **Execution –** Model use relies on analysts entering accurate inputs, accessing the daily refreshed cost of funds file, and being trained to interpret results correctly. The developer noted that the tool prompts for access if the COF file is unavailable and provides error messages instructing users to contact RQCRA. MRO considered these assumptions reasonable, as process controls exist, though continued oversight is warranted to ensure inputs and training remain adequate.

---

✅ This version now gives **each assumption/limitation with developer rationale, measurability, monitoring, and MRO’s stance** — in the same style as the PSI/coverage detail you liked.

Would you like me to **merge these into your IVR template headings** (Methodology, Coverage, Data/Input, Development, Implementation, Model Use, Compensating Measures) so it’s formatted exactly like your screenshots?
