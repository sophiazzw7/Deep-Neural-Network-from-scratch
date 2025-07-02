import numpy as np
import pandas as pd

# ─── 1) Configuration ────────────────────────────────────────────────────────────
dataset = "test_2"
segments = ["auto", "credit_card_consolidation", "home_improvement", "others"]
segment_map = {
    "auto": "Auto",
    "credit_card_consolidation": "Debt Consolidation",
    "home_improvement": "Home Improvement",
    "others": "Others"
}

# ─── 2) Load dev & prod data ────────────────────────────────────────────────────
nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(
    segments, dataset=dataset, subset="all"
)
fi_dict = get_nonprod_fi(segments)
df      = get_prod_data_updated(subset="all")   # full-period prod data

# ─── 3) PSI helper ─────────────────────────────────────────────────────────────
def calculate_psi(expected: np.ndarray,
                  actual:   np.ndarray,
                  buckets: int = 10) -> float:
    if expected.size == 0 or actual.size == 0:
        return np.nan
    qs = np.linspace(0, 100, buckets + 1)
    edges = np.percentile(expected, qs)
    edges[0]  -= 1e-8
    edges[-1] += 1e-8
    exp_cnt, _ = np.histogram(expected, bins=edges)
    act_cnt, _ = np.histogram(actual,   bins=edges)
    exp_pct = exp_cnt / exp_cnt.sum()
    act_pct = act_cnt / act_cnt.sum()
    eps = 1e-8
    exp_pct = np.where(exp_pct == 0, eps, exp_pct)
    act_pct = np.where(act_pct == 0, eps, act_pct)
    return np.sum((act_pct - exp_pct) * np.log(act_pct / exp_pct))


# ─── 4) Compute Shapley-weighted PSI ────────────────────────────────────────────
results = pd.DataFrame(index=[dataset], columns=segments, dtype=float)

for seg in segments:
    model_label = segment_map[seg]
    prod_seg    = df[df["LOANPURPOSECATEGORY"] == model_label]

    # unpack prod DATA into DataFrame
    prod_fe_df = prod_seg["DATA"].apply(pd.Series)

    # unpack dev  DATA into DataFrame (fix for your error!)
    dev_fe_df  = nonprod_fe[seg].apply(pd.Series)

    fi_df      = fi_dict[seg]  # columns: ['feature','importance']

    # intersection of feature names
    common_feats = dev_fe_df.columns.intersection(prod_fe_df.columns)

    psi_list = []
    for feat in common_feats:
        exp_arr = dev_fe_df[feat].dropna().values
        act_arr = prod_fe_df[feat].dropna().values
        psi_val = calculate_psi(exp_arr, act_arr, buckets=10)
        psi_list.append({"feature": feat, "psi": psi_val})

    psi_df = pd.DataFrame(psi_list)
    merged = psi_df.merge(fi_df, on="feature", how="inner")

    # weighted average
    weighted_psi = (merged["psi"] * merged["importance"]).sum() / merged["importance"].sum()
    results.loc[dataset, seg] = weighted_psi

print(results.T)
