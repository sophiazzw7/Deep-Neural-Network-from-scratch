import numpy as np
import pandas as pd

# ——— pure-Python PSI computation ———
def calculate_psi(expected_vals: np.ndarray,
                  actual_vals:   np.ndarray,
                  buckets: int = 10) -> float:
    """
    PSI = sum( (act_pct - exp_pct) * ln(act_pct/exp_pct) )
    Bins defined by quantiles of expected_vals.
    Returns NaN if either input is empty.
    """
    if expected_vals.size == 0 or actual_vals.size == 0:
        return np.nan

    # 1) define edges by expected quantiles
    qs = np.linspace(0, 100, buckets + 1)
    edges = np.percentile(expected_vals, qs)
    edges[0]  -= 1e-8
    edges[-1] += 1e-8

    # 2) histogram counts
    exp_counts, _ = np.histogram(expected_vals, bins=edges)
    act_counts, _ = np.histogram(actual_vals,   bins=edges)

    # 3) proportions
    exp_pct = exp_counts / exp_counts.sum()
    act_pct = act_counts / act_counts.sum()

    # 4) avoid zeros
    eps = 1e-8
    exp_pct = np.where(exp_pct == 0, eps, exp_pct)
    act_pct = np.where(act_pct == 0, eps, act_pct)

    # 5) PSI formula
    psi_values = (act_pct - exp_pct) * np.log(act_pct / exp_pct)
    return psi_values.sum()


# ——— rebuild results DataFrame ———
results = pd.DataFrame(index=[dataset], columns=segments, dtype=float)

for s in segments:
    model_filter = segment_map[s]

    # — production side (actual) —
    prod_seg = df[df['LOANPURPOSECATEGORY'] == model_filter]
    prod_min_scores = prod_seg.groupby('APPID')['SCORE'].min()
    actual_vals = prod_min_scores.values

    # — development side (expected) —
    dev_df = nonprod_scores[s].copy()
    dev_df.columns = ['SCORE']
    expected_vals = dev_df['SCORE'].values

    # compute PSI
    psi_val = calculate_psi(expected_vals, actual_vals, buckets=10)
    results.loc[dataset, s] = psi_val

# show in long form
results.T
