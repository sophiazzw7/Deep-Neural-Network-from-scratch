import pandas as pd
import numpy  as np
import matplotlib.pyplot as plt

# ─── Adjust these imports to match your project ───────────────────────────────
from your_project.data_loader import get_nonprod_data, get_nonprod_fi, get_prod_data_updated
# ────────────────────────────────────────────────────────────────────────────────

# 1) Define your segment mapping ────────────────────────────────────────────────
segment_key   = "credit_card_consolidation"    # internal code
segment_label = "Debt Consolidation"           # what appears in LOANPURPOSE

# 2) Load development data ──────────────────────────────────────────────────────
nonprod_scores, nonprod_features, nonprod_target = get_nonprod_data(
    [segment_key], dataset="test_2", subset="all"
)
fi_dict = get_nonprod_fi([segment_key])

# 3) Build a dev_scores DataFrame with a single column "SCORE" ──────────────────
dev_raw = nonprod_scores[segment_key]
if isinstance(dev_raw, pd.Series):
    dev_scores = dev_raw.to_frame(name="SCORE")
elif hasattr(dev_raw, "__len__") and not isinstance(dev_raw, (int, float)):
    dev_scores = pd.DataFrame(dev_raw, columns=["SCORE"])
else:
    dev_scores = pd.DataFrame([dev_raw], columns=["SCORE"], index=[0])

print("dev_scores shape:", dev_scores.shape)

# 4) Manual PSI function (score‐based) ──────────────────────────────────────────
def manual_psi(expected, actual, bins=20):
    eps = 1e-6
    # common bin edges
    edges = np.linspace(min(expected.min(), actual.min()),
                        max(expected.max(), actual.max()), bins+1)
    exp_cnt, _ = np.histogram(expected, bins=edges)
    act_cnt, _ = np.histogram(actual,   bins=edges)
    exp_pct = exp_cnt / (exp_cnt.sum() + eps) + eps
    act_pct = act_cnt / (act_cnt.sum()   + eps) + eps
    return ((act_pct - exp_pct) * np.log(act_pct/exp_pct)).sum()

# 5) Load production data ──────────────────────────────────────────────────────
prod_all  = get_prod_data_updated("2024-07-01","2024-12-31", subset="all")
prod_auto = get_prod_data_updated("2024-07-01","2024-12-31", subset="auto_approved")

# 6) Compute PSI on the SCORE column ───────────────────────────────────────────
# extract arrays
dev_arr  = dev_scores["SCORE"].values
mask_all = prod_all["LOANPURPOSE"] == segment_label
mask_auto= prod_auto["LOANPURPOSE"] == segment_label

prod_all_arr  = prod_all.loc[mask_all,  "SCORE"].values
prod_auto_arr = prod_auto.loc[mask_auto, "SCORE"].values

psi_full = manual_psi(dev_arr, prod_all_arr,  bins=20)
psi_auto = manual_psi(dev_arr, prod_auto_arr, bins=20)

print(f"\nManual Score-PSI for {segment_label}:")
print(f"  • Full funnel:    {psi_full:.3f}")
print(f"  • Auto-approved:  {psi_auto:.3f}")

# 7) Campaign decision rates ───────────────────────────────────────────────────
camp = prod_all.loc[
    mask_all &
    (pd.to_datetime(prod_all["SCORE_DATE"]) >= "2024-11-01") &
    (pd.to_datetime(prod_all["SCORE_DATE"]) <= "2024-12-31")
]

print(f"\nCampaign decisions for {segment_label} (Nov–Dec 2024):")
print(camp["DECISIONZORS"].value_counts(dropna=False))
print("\nCampaign decision rates:")
print(camp["DECISIONZORS"].value_counts(normalize=True, dropna=False))

# 8) Top Shapley feature drift ─────────────────────────────────────────────────
fi_df    = fi_dict[segment_key]
top_feat = fi_df.sort_values("importance", ascending=False).iloc[0]["feature"]
print(f"\nTop feature by Shapley importance: {top_feat}")

dev_feat_arr  = nonprod_features[segment_key][top_feat].fillna(-1).values
prod_feat_arr = (
    prod_auto.loc[mask_auto, "DATA"]
            .apply(pd.Series)[top_feat]
            .fillna(-1)
            .values
)

print(f"Development mean {top_feat}:  {dev_feat_arr.mean():.1f}")
print(f"Production mean {top_feat}:  {prod_feat_arr.mean():.1f}")

plt.figure(figsize=(8,4))
plt.hist(dev_feat_arr,  bins=50, alpha=0.6, label="Dev",  density=True)
plt.hist(prod_feat_arr, bins=50, alpha=0.6, label="Prod", density=True)
plt.title(f"{top_feat} distribution shift")
plt.legend()
plt.show()
