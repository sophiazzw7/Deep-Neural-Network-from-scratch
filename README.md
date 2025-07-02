import numpy as np
import pandas as pd

# ─── 1) Your existing imports & data prep ───────────────────────────────────────
# (Make sure these functions are already defined in your notebook)
# from zaml.analyze.data_analysis.distribution_drift import PSI   # remove this

segments    = ["auto", "credit_card_consolidation", "home_improvement", "others"]
segment_map = {
    "auto": "Auto",
    "credit_card_consolidation": "Debt Consolidation",
    "home_improvement": "Home Improvement",
    "others": "Others"
}
dataset     = "test_2"

# dev‐period data
nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(
    segments, dataset=dataset, subset="all"
)
# Shapley importances per segment
fi_dict = get_nonprod_fi(segments)

# prod‐period DataFrame (already filtered to auto‐approved earlier)
df = get_prod_data_updated(subset="all")


# ─── 2) PSI helper ─────────────────────────────────────────────────────────────
def psi_for_series(exp: np.ndarray, act: np.ndarray, bins: int = 10) -> float:
    """Population Stability Index between two 1D arrays."""
    if exp.size == 0 or act.size == 0:
        return np.nan

    # define bin edges by expected quantiles
    qs = np.linspace(0, 100, bins + 1)
    edges = np.percentile(exp, qs)
    edges[0]  -= 1e-8
    edges[-1] += 1e-8

    # histogram counts
    exp_cnt, _ = np.histogram(exp, bins=edges)
    act_cnt, _ = np.histogram(act, bins=edges)

    # convert to proportions
    exp_pct = exp_cnt / exp_cnt.sum()
    act_pct = act_cnt / act_cnt.sum()

    # avoid zeros
    eps = 1e-8
    exp_pct = np.where(exp_pct == 0, eps, exp_pct)
    act_pct = np.where(act_pct == 0, eps, act_pct)

    # PSI formula
    return np.sum((act_pct - exp_pct) * np.log(act_pct / exp_pct))


# ─── 3) Compute Shapley‐weighted feature PSI ───────────────────────────────────
results = pd.DataFrame(index=[dataset], columns=segments, dtype=float)

for seg in segments:
    label      = segment_map[seg]
    prod_seg   = df[df["LOANPURPOSECATEGORY"] == label]

    # unpack production features
    prod_fe_df = prod_seg["DATA"].apply(pd.Series)

    # unpack development features
    dev_raw    = nonprod_fe[seg]
    if isinstance(dev_raw, pd.DataFrame):
        dev_fe_df = dev_raw.copy()
    else:
        # Series of dicts or lists → DataFrame
        dev_fe_df = pd.DataFrame(list(dev_raw), index=dev_raw.index)

    # intersect feature names
    common_feats = dev_fe_df.columns.intersection(prod_fe_df.columns)

    # compute PSI per feature
    psi_list = []
    for feat in common_feats:
        exp_vals = dev_fe_df[feat].dropna().values
        act_vals = prod_fe_df[feat].dropna().values
        psi_list.append({
            "feature": feat,
            "psi": psi_for_series(exp_vals, act_vals, bins=10)
        })

    psi_df = pd.DataFrame(psi_list)

    # merge with Shapley importances
    merged = psi_df.merge(fi_dict[seg], on="feature", how="inner")

    # weighted average
    weighted_psi = (merged["psi"] * merged["importance"]).sum() / merged["importance"].sum()
    results.loc[dataset, seg] = weighted_psi

# ─── 4) Display results ───────────────────────────────────────────────────────
print(results.T)
