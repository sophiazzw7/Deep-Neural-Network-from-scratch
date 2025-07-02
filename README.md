import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# ── 1) Helpers & manual PSI ────────────────────────────────────────────────

def manual_psi(expected, actual, bins=20):
    """Compute PSI of actual vs expected arrays using equal-width bins."""
    eps = 1e-6
    lo = min(expected.min(), actual.min())
    hi = max(expected.max(), actual.max())
    edges = np.linspace(lo, hi, bins+1)
    exp_cnt, _ = np.histogram(expected, bins=edges)
    act_cnt, _ = np.histogram(actual,   bins=edges)
    exp_pct = exp_cnt / (exp_cnt.sum()+eps) + eps
    act_pct = act_cnt / (act_cnt.sum()+eps) + eps
    return ((act_pct - exp_pct) * np.log(act_pct/exp_pct)).sum()

# ── 2) Load your data (adjust these imports) ────────────────────────────────

from your_project.data_loader import (
    get_nonprod_data, get_nonprod_fi,
    get_prod_data_updated
)

# dev back-test data
segment_key   = "credit_card_consolidation"
segment_label = "Debt Consolidation"

nonprod_scores, nonprod_feats, _ = get_nonprod_data(
    [segment_key], dataset="test_2", subset="all"
)
fi_dict = get_nonprod_fi([segment_key])

# prod data
prod_all  = get_prod_data_updated("2024-07-01","2024-12-31", subset="all")
prod_app  = get_prod_data_updated("2024-07-01","2024-12-31", subset="approved")

# these come from your earlier cells
# auto_approved_rowids = set(...)            # ROWIDs in prod_all that were auto-approved
# est_auto_approved_zest_keys = set(...)     # ZEST_KEYs in dev back-test that map to auto-approvals

# ── 3) Build the “dev auto-approved” score array ──────────────────────────

# grab the raw dev Series, rename to SCORE
dev_raw = nonprod_scores[segment_key]
dev_df  = dev_raw.to_frame(name="SCORE")

# filter to only those apps flagged auto-approved in your back-test
dev_df_auto = dev_df.loc[ est_auto_approved_zest_keys.intersection(dev_df.index) ]

# if your dev index is applicationid instead of ZEST_KEY, you may need:
# dev_df_auto = dev_df_auto.merge(
#     target_df[['applicationid']], left_index=True, right_index=True
# ).groupby('applicationid')['SCORE'].min().to_frame()

print("Dev auto-approved shape:", dev_df_auto.shape)
print(dev_df_auto.head())

# ── 4) Build the “prod auto-approved” score array ─────────────────────────

prod_df  = prod_app[ prod_app["ROWID"].str.strip().isin(auto_approved_rowids) ]
prod_df  = prod_df[ prod_df["LOANPURPOSE"] == segment_label ]

prod_df_auto = prod_df.groupby('APPID')['SCORE'].min().to_frame()
print("\nProd auto-approved shape:", prod_df_auto.shape)
print(prod_df_auto.head())

# ── 5) Sanity-check histograms ────────────────────────────────────────────

plt.figure(figsize=(8,4))
plt.hist(dev_df_auto['SCORE'],  bins=30, alpha=0.6, label='Dev auto-app', density=True)
plt.hist(prod_df_auto['SCORE'], bins=30, alpha=0.6, label='Prod auto-app', density=True)
plt.title("Auto-approved Score Distribution")
plt.legend()
plt.show()

# ── 6) Compute and compare PSI ─────────────────────────────────────────────

psi_auto = manual_psi(
    dev_df_auto['SCORE'].values,
    prod_df_auto['SCORE'].values,
    bins=20
)
print(f"\nManual auto-approved PSI: {psi_auto:.3f}")

# Compare with their published number (e.g. ~0.116 for DC)
