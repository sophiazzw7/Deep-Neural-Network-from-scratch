import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import re

# ── 0) Config ─────────────────────────────────────────────────────────────────
# adjust paths as needed
COMPILED_FILE = "/d/shared/ogm/prod_data/CompiledZestData_Zest_20250205.csv"
CURR_DECISIONS = "/d/shared/ogm/prod_data/Zest_MDL_CSV4Decision.csv"

segment_key   = "credit_card_consolidation"
segment_label = "Debt Consolidation"

# ── 1) manual PSI helper ──────────────────────────────────────────────────────
def manual_psi(expected, actual, bins=20):
    eps = 1e-6
    lo = min(expected.min(), actual.min())
    hi = max(expected.max(), actual.max())
    edges = np.linspace(lo, hi, bins+1)
    exp_cnt, _ = np.histogram(expected, bins=edges)
    act_cnt, _ = np.histogram(actual,   bins=edges)
    exp_pct = exp_cnt / (exp_cnt.sum()+eps) + eps
    act_pct = act_cnt / (act_cnt.sum()+eps) + eps
    return ((act_pct - exp_pct) * np.log(act_pct/exp_pct)).sum()

# ── 2) pull auto‐approved ROWIDs from the CompiledZestData fixed‐width dump ───
def extract_auto_rowids(compiled_path):
    # 7 header rows then one giant header‐line in column 0, then the data, then 2 trailer rows
    df = pd.read_csv(compiled_path, skiprows=7, header=None, dtype=str)
    raw_header = df.iloc[0, 0]
    data = df.iloc[1:-2, 0].copy()  # drop the header line & trailers
    
    # helper to find start/end in the giant header string
    def find_slice_positions(text, field):
        pos = text.find(field)
        if pos < 0:
            raise KeyError(f"cannot find '{field}' in header")
        tail = text[pos + len(field):]
        m = re.search(r"\w+", tail)
        if not m:
            raise ValueError(f"no value‐delimiter after '{field}'")
        start = pos + len(field)
        end   = start + m.start()
        return start, end

    # fields we need:
    cols = [
        "ZestZamlProcessingId",
        "ExternalApplicationId",
        "RowId",
        "LoanStatus",
        "Auto-Manual_Flag",
        "WasAutoApproved",
        "ApplicationDate"
    ]

    out = {}
    for c in cols:
        s,e = find_slice_positions(raw_header, c)
        out[c] = data.str[s:e].str.strip()

    cols_df = pd.DataFrame(out)
    cols_df["WasAutoApproved"] = pd.to_numeric(cols_df["WasAutoApproved"])
    cols_df["ApplicationDate"]  = pd.to_datetime(cols_df["ApplicationDate"])
    
    # only keep the ones that actually got auto-approved
    return set(cols_df.loc[cols_df["WasAutoApproved"] == 1, "RowId"])

auto_approved_rowids = extract_auto_rowids(COMPILED_FILE)
print("found", len(auto_approved_rowids), "auto-approved ROWIDs")

# ── 3) load dev and prod scores ────────────────────────────────────────────────
from your_project.data_loader import get_nonprod_data, get_nonprod_fi, get_prod_data_updated

# dev back-test
nonprod_scores, nonprod_feats, _ = get_nonprod_data(
    [segment_key], dataset="test_2", subset="all"
)

# production
prod_all  = get_prod_data_updated("2024-07-01","2024-12-31", subset="all")
prod_app  = get_prod_data_updated("2024-07-01","2024-12-31", subset="approved")

# ── 4) slice to auto‐approved DebtConso scores ────────────────────────────────
# DEV
dev_raw = nonprod_scores[segment_key]
dev_df  = pd.DataFrame({"SCORE": dev_raw})
dev_auto = dev_df.loc[dev_df.index.intersection(est_auto_approved_zest_keys)]
print("dev auto shape:", dev_auto.shape)

# PROD
prod_auto = (
    prod_app
    .loc[lambda df: df["ROWID"].str.strip().isin(auto_approved_rowids)]
    .loc[lambda df: df["LOANPURPOSE"] == segment_label]
    .groupby("APPID")["SCORE"]
    .min()
    .to_frame()
)
print("prod auto shape:", prod_auto.shape)

# ── 5) histogram sanity check ─────────────────────────────────────────────────
plt.figure(figsize=(8,4))
plt.hist(dev_auto["SCORE"],  bins=30, alpha=0.6, label="Dev auto-app",  density=True)
plt.hist(prod_auto["SCORE"], bins=30, alpha=0.6, label="Prod auto-app", density=True)
plt.title("Auto-approved Score Distribution\nDebt Consolidation")
plt.legend()
plt.show()

# ── 6) compute manual PSI ─────────────────────────────────────────────────────
psi_auto = manual_psi(
    dev_auto["SCORE"].values,
    prod_auto["SCORE"].values,
    bins=20
)
print(f"Manual auto-approved PSI (DC): {psi_auto:.3f}")
