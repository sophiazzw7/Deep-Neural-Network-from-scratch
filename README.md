Stability Testing

The model’s stability is monitored through two mechanisms:

Robustness Testing: This metric evaluates the reasonableness of outputs under stressed versus baseline scenarios, expressed as the ratio of cumulative losses over 12- and 24-month horizons. It is applied to PS Default, PS Paydown, OPE Default, and OPE Paydown components and is reviewed annually through OGM reporting.

Population Stability Index (PSI): Factor-level PSI metrics are embedded in the model implementation to identify distributional shifts in input variables. A PSI threshold of 0.25 is used, consistent with literature guidance, with higher values signaling potential input data integrity issues.

MRO reviewed the pending 2024Q4 OGM report and confirmed that there were insufficient PAG deals to conduct either robustness testing or PSI-based stability checks. To determine if this limitation was persistent, MRO reviewed three prior reports (2022Q2, 2022Q4, and 2023Q4). These reports likewise reported no PAG deals, and no stability tests were performed. After communication with the model developer, MRO confirmed that there have been no or very limited PAG deals since 2019. Accordingly, stability testing cannot currently be executed.

Backtesting

Backtesting is designed to evaluate the accuracy of the model’s predicted amounts for defaults and paydowns. The metric used is the relative error, 
(
Actual
−
Predicted
)
/
Actual
(Actual−Predicted)/Actual, which is assessed annually across multiple forecast horizons (3, 6, 12, 18, and 24 months). Consecutive breaches would trigger diagnostic reviews, including macroeconomic time series analysis and component-level evaluation.

MRO’s review of the 2024Q4 OGM report showed that insufficient PAG deal volume prevented backtesting. Similar limitations were observed in the three prior OGM reports (2022Q2, 2022Q4, and 2023Q4), which also indicated no PAG deals. MRO therefore confirmed that given the absence of PAG deals since 2019, the backtesting framework cannot be carried out at this time.
