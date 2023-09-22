# Deep-Neural-Network-from-scratch

This is a project where I built a Deep Neural Network to perform Photo classification. 
I build the model from scratch, where I calculated the Forward and Backward Propagation, calculated the loss function and updated parameters.




In the course of validating a model, the developer aimed to demonstrate the rationale behind selecting the current model over a hyperparameter-optimized (hyperopt) counterpart. This evaluation involved a comprehensive assessment of performance metrics, encompassing AUC, precision, F1 score, and several other relevant statistics. While the current model exhibited marginally superior overall performance, it was observed that in specific datasets, the hyperopt model outperformed it. To gain deeper insights into this comparison, two statistical tests were employed: the Delong test and the Mann–Whitney test.

The Delong test results revealed significant disparities between the two models within certain model segments, adding complexity to the analysis. In stark contrast, the Mann–Whitney test produced outcomes that indicated no statistical difference between the models; the results overlapped across all segments. This divergence in results between the two tests prompted a thorough investigation.

Upon deeper examination, the developer discovered that the inconsistent outcomes were not unique to this case but were a recurring challenge when applying both the Delong and Mann–Whitney tests in similar analyses. To shed light on the reasons behind this discrepancy, an illustrative example was presented. The investigation ultimately pointed to the assumptions underpinning these tests.

Notably, the Delong test assumes that the test statistic follows a normal distribution, a requirement that was difficult to validate due to the computational demands associated with simulating numerous model pairs and calculating AUC differences. In response, the developer conducted a rigorous literature review, unearthing multiple warnings against employing the Delong test in nested model contexts. Authors in these references emphasized that the assumption of statistical significance between features and outcomes doesn't always hold, particularly in gradient boosting models such as XGBoost.

The crux of the argument centers on two key points: the stringent nature of the Delong test assumption and the challenge of directly testing this assumption. The developer's literature review supports these assertions, with several references concurring that the Delong test may not be suitable for nested models.

In conclusion, considering the robust literature support provided by the developer, the Model Review Office (MRO) deemed it reasonable not to rely on the Delong test results. The Mann–Whitney test, which indicated no statistically significant difference between the current model and the hyperopt model, further substantiated the selection of the current model. This investigation offers valuable insights into model comparison techniques and the limitations of specific statistical tests, contributing to a well-informed decision-making process.
