import numpy as np
import pandas as pd

# ─── Configuration ─────────────────────────────────────────────────────────────
# (Adjust dates as needed for your production window)
start_date = "2024-02-01"
end_date   = "2024-04-30"
dataset    = "test_2"

# your segments and human-readable map
segments = ["auto", "credit_card_consolidation", "home_improvement", "others"]
segment_map = {
    "auto": "Auto",
    "credit_card_consolidation": "Debt Consolidation",
    "home_improvement": "Home Improvement",
    "others": "Others"
}

# ─── Data Import ────────────────────────────────────────────────────────────────
# dev scores, dev feature-values, dev targets
nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(
    segments, dataset=dataset, subset="all"
)

# feature-importance dict (Shapley importances per segment)
fi_dict = get_nonprod_fi(segments)

# production data (all, we'll filter by LOANPURPOSECATEGORY next)
df = get_prod_data_updated(start_date, end_date, subset="all")


# ─── PSI Helper ─────────────────────────────────────────────────────────────────
def calculate_psi(expected: np.ndarray,
                  actual:   np.ndarray,
                  buckets: int = 10) -> float:
    """
    Compute Population Stability Index between two arrays of scores.
    Bins are defined by quantiles of the 'expected' array.
    Returns np.nan if either array is empty.
    """
    if expected.size == 0 or actual.size == 0:
        return np.nan

    # 1) bin edges by expected quantiles
    qs = np.linspace(0, 100, buckets + 1)
    edges = np.percentile(expected, qs)
    edges[0]  -= 1e-8  # expand first/last edge to catch extremes
    edges[-1] += 1e-8

    # 2) histogram counts
    exp_counts, _ = np.histogram(expected, bins=edges)
    act_counts, _ = np.histogram(actual,   bins=edges)

    # 3) to proportions
    exp_pct = exp_counts / exp_counts.sum()
    act_pct = act_counts / act_counts.sum()

    # 4) avoid zeros for log
    eps = 1e-8
    exp_pct = np.where(exp_pct == 0, eps, exp_pct)
    act_pct = np.where(act_pct == 0, eps, act_pct)

    # 5) PSI formula
    psi_vals = (act_pct - exp_pct) * np.log(act_pct / exp_pct)
    return psi_vals.sum()


# ─── Shapley-Weighted PSI by Segment ────────────────────────────────────────────
results = pd.DataFrame(index=[dataset], columns=segments, dtype=float)

for seg in segments:
    # filter prod data to this segment
    model_label = segment_map[seg]
    prod_seg    = df[df["LOANPURPOSECATEGORY"] == model_label]

    # unpack feature-matrix from the 'DATA' column (assumed dict-like)
    prod_fe_df  = prod_seg["DATA"].apply(pd.Series)

    # dev-period feature values & their importances
    dev_fe_df   = nonprod_fe[seg]
    fi_df       = fi_dict[seg]            # columns: ['feature','importance']

    # only keep features present in both dev & prod
    common_feats = dev_fe_df.columns.intersection(prod_fe_df.columns)

    psi_list = []
    for feat in common_feats:
        exp_arr = dev_fe_df[feat].dropna().values
        act_arr = prod_fe_df[feat].dropna().values
        psi_val = calculate_psi(exp_arr, act_arr, buckets=10)
        psi_list.append({"feature": feat, "psi": psi_val})

    psi_df = pd.DataFrame(psi_list)

    # merge on feature to bring in importance
    merged = psi_df.merge(fi_df, on="feature", how="inner")

    # compute Shapley-weighted PSI
    weighted_psi = (merged["psi"] * merged["importance"]).sum() / merged["importance"].sum()
    results.loc[dataset, seg] = weighted_psi

# display
print(results.T)
