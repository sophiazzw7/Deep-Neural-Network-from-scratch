import numpy as np
import pandas as pd

# ─── 1) Configuration ────────────────────────────────────────────────────────────
segments = ["auto","credit_card_consolidation","home_improvement","others"]
segment_map = {
    "auto": "Auto",
    "credit_card_consolidation": "Debt Consolidation",
    "home_improvement": "Home Improvement",
    "others": "Others"
}
dataset = "test_2"

# ─── 2) Load your data exactly as before ───────────────────────────────────────
nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(
    segments, dataset=dataset, subset="all"
)
fi_dict = get_nonprod_fi(segments)
df      = get_prod_data_updated(subset="all")   # production DataFrame

# ─── 3) Plain-Python PSI function ───────────────────────────────────────────────
def calculate_psi(expected: np.ndarray,
                  actual:   np.ndarray,
                  buckets:  int = 10) -> float:
    """
    Population Stability Index between two numeric arrays.
    Bins by the quantiles of `expected`.
    """
    if expected.size == 0 or actual.size == 0:
        return np.nan

    # 1) define bin edges using expected quantiles
    quantiles = np.linspace(0, 100, buckets + 1)
    edges = np.percentile(expected, quantiles)
    edges[0]  -= 1e-8
    edges[-1] += 1e-8

    # 2) get counts
    exp_cnt, _ = np.histogram(expected, bins=edges)
    act_cnt, _ = np.histogram(actual,   bins=edges)

    # 3) convert to proportions
    exp_pct = exp_cnt / exp_cnt.sum()
    act_pct = act_cnt / act_cnt.sum()

    # 4) avoid zeros
    eps = 1e-8
    exp_pct = np.where(exp_pct == 0, eps, exp_pct)
    act_pct = np.where(act_pct == 0, eps, act_pct)

    # 5) PSI formula
    return np.sum((act_pct - exp_pct) * np.log(act_pct / exp_pct))


# ─── 4) Compute Shapley-Weighted Feature PSI ──────────────────────────────────
results = pd.DataFrame(index=[dataset], columns=segments, dtype=float)

for seg in segments:
    label    = segment_map[seg]
    prod_df  = df[df["LOANPURPOSECATEGORY"] == label]

    # production-side feature matrix
    prod_fe_df = pd.DataFrame(list(prod_df["DATA"]), index=prod_df.index)

    # development-side feature matrix
    dev_raw   = nonprod_fe[seg]
    dev_fe_df = pd.DataFrame(list(dev_raw), index=dev_raw.index)

    # get intersection of feature names
    common_feats = dev_fe_df.columns.intersection(prod_fe_df.columns)

    # compute PSI for each feature
    psi_list = []
    for feat in common_feats:
        exp_vals = dev_fe_df[feat].dropna().values
        act_vals = prod_fe_df[feat].dropna().values
        psi_val  = calculate_psi(exp_vals, act_vals, buckets=10)
        psi_list.append({"feature": feat, "psi": psi_val})

    psi_df = pd.DataFrame(psi_list)

    # merge with Shapley importances and do weighted average
    merged = psi_df.merge(fi_dict[seg], on="feature", how="inner")
    weighted_psi = (merged["psi"] * merged["importance"]).sum() / merged["importance"].sum()

    results.loc[dataset, seg] = weighted_psi

# ─── 5) Display ───────────────────────────────────────────────────────────────
print(results.T)
