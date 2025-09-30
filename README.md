MRO reviewed the model documentation and supporting evidence provided by the developer. The review confirmed that there have been no material changes to the model methodology or assumptions since the last validation. The updated MDD was resubmitted to align with the latest MRO template and to reflect corrected model coverage information.

The model assumes that randomly selected seasoned Sheffield loans are representative of incoming loan pools. The developer acknowledged that collateral variability across deals may create portfolio drift, which cannot be directly measured. PSI testing can help identify drift, but may also generate false positives.

The development dataset spans the 2008–2010 Great Recession but excludes post-March 2020 data, meaning COVID-19 effects are not captured. Additionally, borrower attributes such as term length and product type are missing prior to 2010, and LTV values are missing prior to 2014. These gaps may reduce the ability to fully capture borrower behavior under stress.

The paydown model does not incorporate macroeconomic drivers, as no stable relationship was identified during development. While this simplifies implementation, it may limit responsiveness under deteriorating conditions. The developer acknowledged this limitation, and MRO recommends refreshing the model once sufficient PAG deal data resumes to reassess sensitivity to downturns.

MRO reviewed performance testing documented in the MDD. Stability testing through robustness and PSI checks could not be executed, as no PAG deals were available in recent OGM cycles (2022Q2, 2022Q4, 2023Q4, 2024Q4). Backtesting was also not performed for the same reason.

Sensitivity and scenario testing were completed during development and prior validations. No methodology changes have occurred, and prior results remain the reference point. Stress testing is not applicable given the model’s primary use. Ongoing monitoring likewise could not be performed, and recent OGM reports rated the model as Performing as Intended.
