import pandas as pd
from zaml.analyze.data_analysis.distribution_drift import PSI

# ─── 1) Load development back‐test scores ────────────────────────────────
# this returns a dict of Series, indexed by ZEST_KEY
nonprod_scores, _, _ = get_nonprod_data(
    ["credit_card_consolidation"],
    dataset="test_2",
    subset="all",
)
dev_raw = nonprod_scores["credit_card_consolidation"]

# wrap it into an N×1 DataFrame exactly like they do
dev_df = dev_raw.to_frame(name="SCORE")

# ─── 2) Subset to your back‐test “auto‐approved” keys ─────────────────────
# est_auto_approved_zest_keys comes from their cell [17]
dev_auto = dev_df.loc[ est_auto_approved_zest_keys.intersection(dev_df.index) ]

# ─── 3) Load production “all” and “approved” data ───────────────────────
prod_all  = get_prod_data_updated("2024-07-01","2024-12-31", subset="all")
prod_app  = get_prod_data_updated("2024-07-01","2024-12-31", subset="approved")

# ─── 4) Extract the prod auto‐approved slice ─────────────────────────────
# auto_approved_rowids from their regex parsing in cell [18–19]
mask   = prod_app["ROWID"].str.strip().isin(auto_approved_rowids)
mask  &= prod_app["LOANPURPOSE"] == "Debt Consolidation"
prod_auto = (
    prod_app[mask]
    .groupby("APPID")["SCORE"]
    .min()                  # pick the lowest of any duplicate scores
    .to_frame(name="SCORE")
)

# ─── 5) Compute PSI exactly like they do ────────────────────────────────
psi = PSI()                  # default is equal‐pop bins (deciles)

# fit on dev_auto (their “fit(nonprod_score_df)” step)
psi.fit(dev_auto)

# transform on prod_auto (their “transform(score_data)” step)
psi_auto = psi.transform(prod_auto)

print("Their auto‐approved PSI: ", psi_auto.iloc[0])
