MOD13512 is a supervised machine learning model that predicts the likelihood of borrower default using XGBoost with a log-loss objective. The model targets defaults defined as loans reaching 60+ days delinquency within an 18-month window, using performance data from funded applicants.

The development team evaluated alternative methods, including logistic regression, naïve Bayes, random forest, and support vector machines. XGBoost was selected for its superior handling of complex data structures and strong predictive performance. Its selection is supported by industry literature (e.g., Schapire, 2003) and aligns with best practices for binary classification in credit risk modeling.

The approach also includes gain-based recursive feature elimination and segment-level sub-models to capture heterogeneity across loan purposes. Overall, the methodology is well-supported, data-driven, and appropriate for the model’s objective.
