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
# dev scores, fe, target
nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(
    segments, dataset=dataset, subset="all"
)

# dev feature‐importance
fi_dict = get_nonprod_fi(segments)

# production data (full period)
df = get_prod_data_updated(subset="all")   # <-- no start/end dates here

# ─── 3) PSI helper ─────────────────────────────────────────────────────────────
def calculate_psi(expected: np.ndarray,
                  actual:   np.ndarray,
                  buckets: int = 10) -> float:
    """
    Population Stability Index between two arrays.
    Bins are defined by quantiles of `expected`.
    """
    if expected.size == 0 or actual.size == 0:
        return np.nan

    # 1) bin edges by expected quantiles
    qs = np.linspace(0, 100, buckets + 1)
    edges = np.percentile(expected, qs)
    edges[0]  -= 1e-8
    edges[-1] += 1e-8

    # 2) counts
    exp_cnt, _ = np.histogram(expected, bins=edges)
    act_cnt, _ = np.histogram(actual,   bins=edges)

    # 3) to proportions
    exp_pct = exp_cnt / exp_cnt.sum()
    act_pct = act_cnt / act_cnt.sum()

    # 4) avoid zeros
    eps = 1e-8
    exp_pct = np.where(exp_pct == 0, eps, exp_pct)
    act_pct = np.where(act_pct == 0, eps, act_pct)

    # 5) PSI formula
    psi_vals = (act_pct - exp_pct) * np.log(act_pct / exp_pct)
    return psi_vals.sum()


# ─── 4) Compute Shapley‐weighted PSI per segment ────────────────────────────────
results = pd.DataFrame(index=[dataset], columns=segments, dtype=float)

for seg in segments:
    model_label = segment_map[seg]
    prod_seg    = df[df["LOANPURPOSECATEGORY"] == model_label]

    # expand the 'DATA' dict‐column into a DataFrame of feature‐values
    prod_fe_df = prod_seg["DATA"].apply(pd.Series)

    # dev‐period features & importances
    dev_fe_df = nonprod_fe[seg]
    fi_df     = fi_dict[seg]          # columns: ['feature','importance']

    # only keep features present in both sets
    common_feats = dev_fe_df.columns.intersection(prod_fe_df.columns)

    # compute raw PSI per feature
    psi_list = []
    for feat in common_feats:
        exp_arr = dev_fe_df[feat].dropna().values
        act_arr = prod_fe_df[feat].dropna().values
        psi_val = calculate_psi(exp_arr, act_arr, buckets=10)
        psi_list.append({"feature": feat, "psi": psi_val})

    psi_df = pd.DataFrame(psi_list)

    # merge with Shapley importances and take weighted average
    merged = psi_df.merge(fi_df, on="feature", how="inner")
    weighted_psi = (merged["psi"] * merged["importance"]).sum() / merged["importance"].sum()

    results.loc[dataset, seg] = weighted_psi

# ─── 5) Show results ────────────────────────────────────────────────────────────
print(results.T)
