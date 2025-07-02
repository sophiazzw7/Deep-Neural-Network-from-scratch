import pandas as pd
from zaml.analyze.data_analysis.distribution_drift import PSI
# adjust these imports to your codebase
from your_project.data_loader import get_nonprod_data, get_prod_data_updated

# Load development scores
nonprod_scores, nonprod_features, _ = get_nonprod_data(
    ["credit_card_consolidation"], dataset="test_2", subset="all"
)

# Build dev_scores_dc robustly
dev_raw = nonprod_scores["credit_card_consolidation"]
if isinstance(dev_raw, pd.Series):
    dev_scores_dc = dev_raw.to_frame(name="SCORE")
elif hasattr(dev_raw, "__len__") and not isinstance(dev_raw, (int, float)):
    dev_scores_dc = pd.DataFrame(dev_raw, columns=["SCORE"])
else:
    dev_scores_dc = pd.DataFrame([dev_raw], columns=["SCORE"], index=[0])

# Fit PSI
psi = PSI()
psi.fit(dev_scores_dc)

# Example: compute full‐funnel PSI for Debt Consolidation
prod_all = get_prod_data_updated("2024-07-01", "2024-12-31", subset="all")
mask_dc   = prod_all["LOANPURPOSE"] == "Debt Consolidation"
dc_scores = prod_all.loc[mask_dc, ["SCORE"]]
psi_full  = psi.transform(dc_scores).iloc[0]
print("Debt Conso full-funnel PSI:", psi_full)

