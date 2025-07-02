import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# -------------- imports (adjust to your codebase) ------------------
from your_project.data_loader import get_nonprod_data, get_nonprod_fi, get_prod_data_updated

# -------------- manual PSI function --------------------------------
def manual_psi(expected, actual, bins=10):
    """Compute PSI of actual vs expected arrays using equal-width bins."""
    eps = 1e-6
    # build common bins
    breakpoints = np.linspace(min(expected.min(), actual.min()),
                              max(expected.max(), actual.max()), bins + 1)
    # get distributions
    exp_counts, _ = np.histogram(expected, bins=breakpoints)
    act_counts, _ = np.histogram(actual,   bins=breakpoints)
    # convert to percentages
    exp_pct = exp_counts / (exp_counts.sum() + eps) + eps
    act_pct = act_counts / (act_counts.sum()   + eps) + eps
    # PSI formula
    psi_vals = (act_pct - exp_pct) * np.log(act_pct / exp_pct)
    return psi_vals.sum()

# -------------- load data ---------------------------------------------------
# development (nonprod) for Debt Consolidation
segments = ["credit_card_consolidation"]
dev_scores, dev_feats, _ = get_nonprod_data(segments, dataset="test_2", subset="all")
fi_dict = get_nonprod_fi(segments)

# production: full funnel and auto-approved
prod_all  = get_prod_data_updated("2024-07-01","2024-12-31", subset="all")
prod_auto = get_prod_data_updated("2024-07-01","2024-12-31", subset="auto_approved")

# -------------- 1) Campaign decision rates -------------------------------
segment_name = "Debt Consolidation"
mask = prod_all["LOANPURPOSE"] == segment_name

camp = prod_all.loc[
    mask & 
    (pd.to_datetime(prod_all["SCORE_DATE"]) >= "2024-11-01") &
    (pd.to_datetime(prod_all["SCORE_DATE"]) <= "2024-12-31"),
    :
]

print("=== Campaign (Nov–Dec) Decisions ===")
print(camp["DECISIONZORS"].value_counts(dropna=False), "\n")
print("=== Campaign Decision Rates ===")
print(camp["DECISIONZORS"].value_counts(normalize=True, dropna=False), "\n")

# -------------- 2) Manual PSI on scores -----------------------------------
# extract score arrays
dev_score_arr  = dev_scores[segment_name].values
prod_score_all = prod_all.loc[mask, "SCORE"].values
prod_score_auto= prod_auto.loc[prod_auto["LOANPURPOSE"]==segment_name, "SCORE"].values

psi_full = manual_psi(dev_score_arr, prod_score_all, bins=20)
psi_auto = manual_psi(dev_score_arr, prod_score_auto, bins=20)
print(f"Manual PSI on SCORES → Full funnel: {psi_full:.3f}, Auto-approved: {psi_auto:.3f}\n")

# -------------- 3) Top-feature drift ---------------------------------------
fi_df     = fi_dict[segment_name]
top_feat  = fi_df.sort_values("importance", ascending=False).iloc[0]["feature"]
print("Top Shapley feature:", top_feat)

# pull feature values
dev_feat_arr  = dev_feats[segment_name][top_feat].fillna(-1).values
prod_feat_arr = (
    prod_auto[prod_auto["LOANPURPOSE"]==segment_name]["DATA"]
    .apply(pd.Series)[top_feat]
    .fillna(-1)
    .values
)

print(f"Development mean {top_feat}:  {dev_feat_arr.mean():.1f}")
print(f"Prod (auto-app) mean {top_feat}: {prod_feat_arr.mean():.1f}\n")

# histogram
plt.figure(figsize=(8,4))
plt.hist(dev_feat_arr,  bins=50, alpha=0.6, label="Dev", density=True)
plt.hist(prod_feat_arr, bins=50, alpha=0.6, label="Prod (auto-app)", density=True)
plt.title(f"Drift in '{top_feat}'")
plt.xlabel(top_feat)
plt.ylabel("Density")
plt.legend()
plt.show()
