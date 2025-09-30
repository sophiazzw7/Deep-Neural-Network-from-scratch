Methodology

The MDD describes the use of a discrete-time proportional odds hazard model estimated through logistic regression. Default and paydown are modeled separately for the Power Sports and Outdoor Power Equipment portfolios. Key variables include origination term length, loan-to-value ratio, product category, Vantage Score 4.0, and TransUnion Bankruptcy Score, with vintage indicators used to address data reliability in earlier periods. Alternative approaches such as Cox proportional hazards and roll-rate frameworks were reviewed but not adopted. The methodology section of the MDD is consistent with industry practice and clearly documents the estimation structure, inputs, and rationale.

Assumptions

The MDD identifies key assumptions tied to the discrete-time hazard framework. These include the proportional odds specification, treatment of censoring, and reliance on origination and credit bureau variables as primary drivers. The documentation also notes that macroeconomic factors were tested but found to have limited incremental impact, and that PSI monitoring could not be performed given the absence of PAG deals. Assumptions are explicitly stated, and where limitations exist, compensating factors (such as vintage indicators) are noted.

Performance Testing

Performance testing was not performed on production data due to the absence of PAG deals in the 2022Q2, 2022Q4, 2023Q4, and 2024Q4 OGM periods. The MDD instead references development testing, including goodness-of-fit comparisons of predicted versus observed hazard rates for both default and paydown models. These results are presented graphically and show alignment between model predictions and historical data. MRO notes that ongoing backtesting, sensitivity testing, and PSI monitoring could not be conducted without PAG volume.
