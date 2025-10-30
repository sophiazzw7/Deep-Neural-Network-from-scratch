Subject: Request for Model Code and Clarifications – Quantitative Model & OGM
Hi [Developer’s Name],
Hope you’re doing well. As part of our ongoing validation review, we noted that the quantitative model code (e.g., Monte Carlo simulation, baseline logic, and distribution-fitting) is not currently included in the SAS folder.
If possible, could you please share the code first, since reviewing it directly would help streamline our follow-up questions and avoid delays in clarification?
Attached are the detailed questions covering both the quantitative model and the OGM implementation.

Quantitative Model Questions
1. Quantitative model codes
* Could you share the quantitative model codes e.g. data prep specific to LDA, distribution fitting, the Monte Carlo (MC) simulation, etc.?
* Please confirm where and how the baseline percentile is defined within the code.

2. Distribution Fitting
* Could you provide the code used to fit the loss frequency and severity distributions and assess the goodness of fit(e.g., Q-Q plots, KS test, AIC comparison)?
* Could you provide the code where the alternative distributions are tested?

3. Parameter Confidence Intervals and Stability
* Have you computed confidence intervals (CIs) for the fitted parameters?
* Have you conducted any parameter stability tests, e.g. the distribution of the parameter?

4. Sensitivity Analysis
* Have you conducted a sensitivity test to assess e.g. how model output (total loss) changes if fitted parameters vary by ±5%?
* If not, could you estimate how such parameter shifts would affect the aggregate forecast (to gauge model sensitivity to misfit)?

5. Stress Percentile 
* How was the stress percentile determined based on the baseline percentile? Was there a documented methodology or decision rule used to select the stress percentile?
* Who explicitly approved this choice, and was it reviewed or signed off by any governance body?
* Were any manual treatments or adjustments applied to the baseline percentile (e.g., outlier exclusion, data smoothing)? How do you handle cased where there is one extreme value and there rest 8 quarters have small loss value?

6. Temporal Consistency of Inputs 
* Have you checked whether the input distributions remain consistent over time?
* Have you considered whether differences across time periods , for example, stressed economic conditions or structural shifts such as pre- and post-merger periods, would still be appropriately represented by the same fitted distribution?
7. Model risk tier
Could you please confirm the current model coverage used to determine the risk tier, for example, specify the approximate exposure or balance sheet amount represented by the model, or provide the rationale for why it is categorized as “High” in ServiceNow?
Additionally, in the MDD risk tier support table, please update Model Reliance to reflect “High”, since the model is actively used for capital and loss estimation.
Could you help us understand why the model tier was overridden to 2 in SNOW? Was it based on provisos model’s tier?


OGM-Related Questions
1. Metrics and Sufficiency
* Both OGM metrics KS and DDT are both based on the Kolmogorov–Smirnov framework and will most likely move in the same direction when distributions change.
* We would suggest adding an additional stability metric, such as a confidence-interval overlap test, that compares 95% confidence intervals of mean and dispersion parameters between development and OGM datasets?

2. KS Threshold Rationale
* The current KS threshold approach (“mean ± 4σ from bootstrap KS”) may lack a clear statistical grounding. Since the KS statistic approximately follows a normal distribution, would it be more appropriate to use ±2σ (corresponding to ~95% confidence) instead of ±4σ (~99.99%) as the decision boundary? 


3. DDT Thresholds
* The documentation explains the Development Data Divergence Test (DDT) conceptually, but no quantitative thresholds are shown.


4. Bootstrap Setup and Implementation
* For the 1,000 bootstrap samples, should each replicate have N equal to the OGM sample size, to ensure the test statistic reflects the actual sampling variability in each OGM period?
* When calculating KS for each replicate, can you confirm whether the comparison is made between the theoretical CDF and the bootstrap sample CDF?
* In other words, is this metric asking “ would it be possible to draw the OGM data directly from the model’s fitted CDF”?

5. Multiple Test Clarification
* Given there are six KS tests (frequency and severity across ET2, ET3, ET7), how are results aggregated at the model level?


Thank you very much for taking the time to review these. Please feel free to send the code first, and we can follow up on the specific questions as needed once we’ve had a chance to review it.
Best regards, Phoebe Chen Model Risk Oversight
