Got it — I’ll keep it very simple, only using what is clearly shown in the 21 photos you uploaded. No extra interpretation. Here’s the pared-down version with straightforward sentences and logical flow:

---

# Methodology

The Sheffield PAG model uses a survival analysis framework to estimate default and paydown rates. The structure is a discrete-time proportional odds hazard model estimated through logistic regression.

Survival analysis was selected because it models the waiting time until default or paydown. The data are monthly, so a discrete-time approach was appropriate. Observations are treated as Bernoulli trials, and hazard rates are conditional probabilities given survival up to that time.

Alternative approaches, such as Cox proportional hazards and parametric hazard models, were reviewed but not adopted. A roll-rate framework was also considered but determined to be less flexible.

Default and paydown are modeled separately for the Power Sports and Outdoor Power Equipment portfolios. Key variables include origination term length, loan-to-value ratio, product category, Vantage Score 4.0, and TransUnion Bankruptcy Score. Vintage indicators were introduced to address unreliable data for certain time periods.

For default estimation, weighted log-odds were used with significant predictors from acquisition and origination variables. For paydown estimation, the models included time on books, origination variables, and acquisition scores. Macroeconomic factors such as unemployment rate were tested but did not materially improve accuracy.

Loss given default was estimated separately by decomposing into roll rates and recovery rates.

Model fit was evaluated by comparing observed and predicted hazard rates, shown in charts for both portfolios. Statistical validity testing was conducted for logistic regression assumptions, with compensating measures documented where direct testing was limited.

---

Would you like me to also shorten this further into a **half-page summary version** (just a few core sentences), so you have both a detailed and a compact option?
