import pandas as pd
import numpy as np
from zaml.analyze.data_analysis.distribution_drift import PSI

# 1) grab your dev back‐test “score” object
nonprod_scores, _, _ = get_nonprod_data(
    ["credit_card_consolidation"],
    dataset="test_2",
    subset="all",
)
dev_raw = nonprod_scores["credit_card_consolidation"]

# 2) turn it into an N×1 DataFrame called "SCORE"
if isinstance(dev_raw, pd.Series):
    dev_df = dev_raw.to_frame("SCORE")
elif isinstance(dev_raw, np.ndarray):
    dev_df = pd.DataFrame(dev_raw, columns=["SCORE"])
else:
    # if it’s a list or something list‐like
    dev_df = pd.DataFrame(list(dev_raw), columns=["SCORE"])

print("dev_df shape:", dev_df.shape)
print(dev_df.head())

# 3) filter to only your back‐test auto‐approved keys
dev_auto = dev_df.loc[ est_auto_approved_zest_keys.intersection(dev_df.index) ]
print("dev_auto shape:", dev_auto.shape)

# 4) load prod “approved” and extract auto‐approved slice
prod_app = get_prod_data_updated("2024-07-01","2024-12-31", subset="approved")

mask = (
    prod_app["ROWID"].str.strip().isin(auto_approved_rowids)
    & (prod_app["LOANPURPOSE"] == "Debt Consolidation")
)
prod_auto = (
    prod_app[mask]
    .groupby("APPID")["SCORE"]
    .min()
    .to_frame(name="SCORE")
)
print("prod_auto shape:", prod_auto.shape)

# 5) compute PSI exactly as their notebook does (decile bins)
psi = PSI()
psi.fit(dev_auto)              # fit on the dev auto‐approved distribution
out = psi.transform(prod_auto) # transform on the prod auto‐approved distribution

print("Auto‐approved PSI:", out.iloc[0])
