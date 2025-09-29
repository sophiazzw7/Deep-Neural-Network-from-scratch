Methodology

The Sheffield PAG model applies a survival analysis framework to estimate default and paydown rates. The approach is based on a discrete-time proportional odds hazard model estimated through logistic regression. Survival analysis was chosen because it allows events such as default or paydown to be modeled as waiting times until occurrence, while accounting for censoring and incorporating explanatory variables.

Two general survival analysis approaches were considered: continuous-time and discrete-time. Continuous-time models, such as the Cox proportional hazards model, estimate hazard rates at any point in time but face challenges with tied events and can produce poor fits when parametric assumptions are applied. Discrete-time models provide hazard rates at specific time intervals. This was more suitable for the Sheffield portfolio because the available data were reported at a monthly level, and the discrete-time framework aligned directly with that granularity. Logistic regression further offered transparency and ease of implementation.

Within this framework, observations are treated as Bernoulli trials with events coded as binary outcomes. Hazard rates are interpreted as conditional probabilities given survival up to that time period. The proportional odds formulation allows predictors to be applied consistently across time, while baseline hazard terms capture variation at each interval. By applying default and paydown hazard rates to beginning balances, the model estimates monthly amounts of default dollars and paydowns, with surviving balances rolled forward into the next period.

Alternative methodologies were also reviewed. Top-down approaches, such as time series and roll rate models, estimate losses at the portfolio level but lack the ability to incorporate borrower-level risk drivers. Bottom-up approaches, such as Markov chain models, require estimation of many transition probabilities, which complicates development. In comparison, the discrete-time proportional odds framework requires fewer transitions, simplifies variable selection, and provides consistent probability estimates across the loan life.

The model was developed separately for the Power Sports and Outdoor Power Equipment portfolios to account for differences in borrower characteristics and data limitations. Each portfolio’s model included origination variables such as term length and loan-to-value ratio, as well as credit risk indicators like Vantage Score 4.0 and TransUnion Bankruptcy Score. Historical data quality issues were addressed by introducing vintage indicators for earlier periods where product, term, or loan-to-value data were missing or unreliable.

For default estimation, the model specification included weighted log-odds with significant contributions from acquisition scores, bankruptcy scores, and loan characteristics. For paydown estimation, the model combined time on books with origination variables and credit scores. In both cases, macroeconomic variables were tested but did not provide meaningful improvement to predictive accuracy. The final specifications were validated against observed hazard rates, with model fits presented graphically to demonstrate alignment.

Loss given default was estimated separately by decomposing it into roll rates and recovery rates. This decomposition allowed more precise measurement of severity and improved the connection between default events and resulting losses.

Statistical validity testing was performed to confirm that logistic regression assumptions were satisfied. Assumptions such as linearity of the logit and independence of predictors were considered, and compensating measures were documented where direct testing was limited. Overall, the discrete-time proportional odds hazard model provided a consistent and transparent structure for forecasting Sheffield PAG default and paydown behavior.

Do you want me to also condense this into a shorter summary version (like a paragraph you could use in an executive section), or keep it only in this detailed report style?

You said:
be simple and only say things that you have firm proof
ChatGPT said:

Got it — I’ll keep it very simple, only using what is clearly shown in the 21 photos you uploaded. No extra interpretation. Here’s the pared-down version with straightforward sentences and logical flow:

Methodology

The Sheffield PAG model uses a survival analysis framework to estimate default and paydown rates. The structure is a discrete-time proportional odds hazard model estimated through logistic regression.

Survival analysis was selected because it models the waiting time until default or paydown. The data are monthly, so a discrete-time approach was appropriate. Observations are treated as Bernoulli trials, and hazard rates are conditional probabilities given survival up to that time.

Alternative approaches, such as Cox proportional hazards and parametric hazard models, were reviewed but not adopted. A roll-rate framework was also considered but determined to be less flexible.

Default and paydown are modeled separately for the Power Sports and Outdoor Power Equipment portfolios. Key variables include origination term length, loan-to-value ratio, product category, Vantage Score 4.0, and TransUnion Bankruptcy Score. Vintage indicators were introduced to address unreliable data for certain time periods.

For default estimation, weighted log-odds were used with significant predictors from acquisition and origination variables. For paydown estimation, the models included time on books, origination variables, and acquisition scores. Macroeconomic factors such as unemployment rate were tested but did not materially improve accuracy.

Loss given default was estimated separately by decomposing into roll rates and recovery rates.

Model fit was evaluated by comparing observed and predicted hazard rates, shown in charts for both portfolios. Statistical validity testing was conducted for logistic regression assumptions, with compensating measures documented where direct testing was limited.
