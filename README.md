The model was built when defaults were low, so it now underpredicts risk as defaults have roughly doubled. The developer proposed a monitoring-only recalibration—adjusting predicted defaults for reporting to reflect current conditions. It doesn’t change the production model, just makes monitoring fairer given today’s higher baseline.


Quantitative Testing Status
Developer testing is limited, as they consider it a non-traditional model. I requested:


Parameter Stability: add confidence intervals and cross-period checks.


Sensitivity: assess total loss impact from ±5% parameter shifts.


Stress Percentile: clarify selection, approval, and outlier treatment.

