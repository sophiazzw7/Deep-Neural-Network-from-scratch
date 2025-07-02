import numpy as np
import pandas as pd

def calculate_psi(expected, actual, buckets=10):
    """
    Calculate PSI between two arrays of scores.
    Bins are defined by quantiles of the expected array.
    """
    if len(expected) == 0 or len(actual) == 0:
        return np.nan

    # Define bin edges by quantiles of the expected (dev) scores
    quantiles = np.linspace(0, 100, buckets + 1)
    bin_edges = np.percentile(expected, quantiles)
    bin_edges[0] -= 1e-8
    bin_edges[-1] += 1e-8

    exp_hist, _ = np.histogram(expected, bins=bin_edges)
    act_hist, _ = np.histogram(actual,   bins=bin_edges)

    exp_pct = exp_hist / exp_hist.sum() if exp_hist.sum() > 0 else np.full_like(exp_hist, 1/len(exp_hist))
    act_pct = act_hist / act_hist.sum() if act_hist.sum() > 0 else np.full_like(act_hist, 1/len(act_hist))

    # Avoid log(0)
    eps = 1e-8
    exp_pct = np.where(exp_pct == 0, eps, exp_pct)
    act_pct = np.where(act_pct == 0, eps, act_pct)

    psi = np.sum((act_pct - exp_pct) * np.log(act_pct / exp_pct))
    return psi

results = pd.DataFrame(index=["test_2"], columns=segments)

for s in segments:
    model_segment_filter = segment_map[s]
    
    # Production period (actual)
    prod_seg = df[df['LOANPURPOSECATEGORY'] == model_segment_filter]
    prod_scores = prod_seg.groupby('APPID')['SCORE'].min()  # Use min if multiple scores per APPID
    
    # Development period (expected)
    dev_df = nonprod_scores[s].copy()
    dev_df.columns = ['SCORE']  # just in case
    dev_scores = dev_df['SCORE']

    # Align indices (use intersection of APPID/index if needed; otherwise just values)
    actual_scores = prod_scores.values
    expected_scores = dev_scores.values

    psi_value = calculate_psi(expected_scores, actual_scores, buckets=10)
    results.loc["test_2", s] = psi_value

print(results.T)
