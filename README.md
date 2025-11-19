Hi Cheng, thank you. 
Just to clarify, the simulation-based quantile comparison we ran was not intended as a parameter-uncertainty bootstrap. We used it as a posterior predictive diagnostic, meaning we simulated repeated datasets from the fitted model and checked whether the observed quantiles look typical under that model. This is the same idea as posterior predictive checks commonly used in the model-assessment literature (e.g., Gelman et al., 1996), where the goal is to evaluate how well a fitted model reproduces key empirical features of the data under repeated sampling.

The AD goodness‐of‐fit test using 10,000 bootstrap replications yielded an observed statistic of 7.79 with a bootstrap p-value of approximately 0.0002.
