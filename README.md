The MOD13512 model is developed to support credit decisioning by classifying applicants as either likely “good” or “bad” based on their probability of default. It uses a supervised machine learning algorithm, XGBoost, to estimate the likelihood that an applicant will become 60 or more days delinquent within an 18-month performance window.

Target and Data:
The model was trained using historical default events from funded applicants—defined as those who completed the loan process and received disbursed funds. Since default outcomes are only observable for funded loans, unfunded applicants were assigned outcomes through reject inference. The model's target is a binary flag indicating whether a funded borrower reached the default threshold.

Segmentation:
To address variation across loan types, the model is segmented into four sub-models corresponding to major loan-purpose groups. This segmentation reflects the developers’ view that self-reported loan purposes may affect risk characteristics and should be modeled independently.

Methodology and Alternatives:
Several modeling approaches were evaluated, including logistic regression, naïve Bayes, random forest, and support vector machines. XGBoost was selected for its strong performance and ability to handle large, complex datasets. The model uses a log-loss objective function appropriate for binary classification problems like credit risk prediction.

Feature Selection:
Features were selected through a recursive feature elimination process using gain-based importance scores. This approach efficiently narrowed down the most relevant predictors and is consistent with industry practices for feature selection in machine learning.

Conclusion:
The use of XGBoost, combined with gain-based feature selection and segment-specific models, is appropriate for the objective of predicting borrower creditworthiness. The methodology is consistent with industry standards and well-aligned with the model’s intended purpose.
