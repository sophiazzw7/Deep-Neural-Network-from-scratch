import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# ── 1) Helpers ───────────────────────────────────────────────────────────────

def to_dataframe(x, name="SCORE"):
    """Coerce a pd.Series / np.ndarray / list into an N×1 DataFrame."""
    if isinstance(x, pd.Series):
        return x.to_frame(name)
    elif isinstance(x, np.ndarray):
        return pd.DataFrame(x, columns=[name])
    else:
        return pd.DataFrame(list(x), columns=[name])

def manual_psi(expected, actual, bins=20):
    """Compute PSI of actual vs expected arrays using equal-width bins."""
    eps = 1e-6
    lo = min(expected.min(), actual.min())
    hi = max(expected.max(), actual.max())
    edges = np.linspace(lo, hi, bins+1)
    exp_cnt, _ = np.histogram(expected, bins=edges)
    act_cnt, _ = np.histogram(actual,   bins=edges)
    exp_pct = exp_cnt / (exp_cnt.sum() + eps) + eps
    act_pct = act_cnt / (act_cnt.sum()   + eps) + eps
    return ((act_pct - exp_pct) * np.log(act_pct/exp_pct)).sum()


# ── 2) Data imports — adjust the path to your project’s loader ──────────────
from your_project.data_loader import (
    get_nonprod_data,
    get_nonprod_fi,
    get_prod_data_updated
)

# define segments & date window
segments      = ["auto","credit_card_consolidation","home_improvement","others"]
segment_label = "Debt Consolidation"
start_date, end_date = "2024-07-01","2024-12-31"

# — 2a) DEV back-test data
nonprod_scores, nonprod_features, nonprod_target = get_nonprod_data(
    segments, dataset="test_2", subset="all"
)
fi_dict = get_nonprod_fi(segments)

# — 2b) PROD OGM data
prod_all = get_prod_data_updated(start_date, end_date, subset="all")
prod_app = get_prod_data_updated(start_date, end_date, subset="approved")


# ── 3) Recompute your “auto-approved” key sets ───────────────────────────────

# 3a) From your “current strategy” CSV → applicationIds
test2 = pd.read_csv("/d/shared/ogm/prod_data/Zest_MDL_CSV4Decision.csv", sep="|")
conds = [
    test2["auto_manual_decision"]=="AutoDeclined",
    test2["auto_manual_decision"]=="AutoApproval",
    test2["auto_manual_decision"]=="ManualW"
]
#  0 = declined, 1 = approved, 0 = manual
test2["auto_approved"] = np.select(conds, [0,1,0], default=np.nan)
auto_approved_appids = set(test2.loc[test2["auto_approved"]==1, "applicationid"])

# 3b) Map those appids back to your dev ZEST_KEYs
tar = pd.read_feather("/d/shared/truist/shared_data/processed/target/target_processed.feather")
tar["applicationid"] = pd.to_numeric(tar["applicationid"])
est_auto_approved_zest_keys = set(
    tar.loc[tar["applicationid"].isin(auto_approved_appids), "ZEST_KEY"]
)

# 3c) From your “CompiledZestData” golden CSV → ROWIDs
import re

def find_substring_start_end(text, substring):
    pos = text.find(substring)
    if pos < 0: return None
    tail = text[pos + len(substring):]
    m = re.search(r"\w+", tail)
    if not m: return None
    return pos + len(substring), pos + len(substring) + m.start()

larry = pd.read_csv(
    "/d/shared/ogm/prod_data/CompiledZestData_Zest_20250205.csv",
    skiprows=7, header=None
).drop(0).iloc[:-2]

# extract each field by slicing on that single raw line
for col in ["ZestZamlProcessingId","ExternalApplicationId",
            "RowId","LoanStatus","Auto-Manual_Flag",
            "WasAutoApproved","ApplicationDate"]:
    start_end = find_substring_start_end(larry.columns[0], col)
    if start_end is None: 
        raise RuntimeError(f"Couldn't parse {col}")
    s,e = start_end
    larry[col] = larry.iloc[:,0].str[s:e].str.strip()

larry["WasAutoApproved"] = pd.to_numeric(larry["WasAutoApproved"])
larry["ApplicationDate"]  = pd.to_datetime(larry["ApplicationDate"])
auto_approved_rowids = set(
    larry.loc[larry["WasAutoApproved"]==1, "RowId"]
)


# ── 4) Build “dev auto-approved” score array ────────────────────────────────
dev_raw  = nonprod_scores["credit_card_consolidation"]
dev_df   = to_dataframe(dev_raw, name="SCORE")

dev_df_auto = dev_df.loc[
    dev_df.index.intersection(est_auto_approved_zest_keys)
]
print("dev_df_auto.shape:", dev_df_auto.shape)


# ── 5) Build “prod auto-approved” score array ───────────────────────────────
prod_df = prod_app[
    prod_app["ROWID"].str.strip().isin(auto_approved_rowids)
]
prod_df = prod_df[prod_df["LOANPURPOSE"] == segment_label]

prod_df_auto = prod_df.groupby("APPID")["SCORE"].min().to_frame()
print("prod_df_auto.shape:", prod_df_auto.shape)


# ── 6) Histogram sanity check ───────────────────────────────────────────────
plt.figure(figsize=(8,4))
plt.hist(dev_df_auto["SCORE"],  bins=30, alpha=0.6, label="Dev auto-app", density=True)
plt.hist(prod_df_auto["SCORE"], bins=30, alpha=0.6, label="Prod auto-app", density=True)
plt.title("Auto-approved Score Distribution")
plt.legend()
plt.show()


# ── 7) Compute manual PSI ───────────────────────────────────────────────────
psi_auto = manual_psi(
    dev_df_auto["SCORE"].values,
    prod_df_auto["SCORE"].values,
    bins=20
)
print(f"Manual auto-approved PSI = {psi_auto:.3f}")
