import numpy as np

def calculate_psi(expected, actual, buckets=10):
    """
    Calculate PSI between two 1D arrays:
      PSI = sum( (A_i - E_i) * ln(A_i/E_i) )
    where E_i and A_i are the % of expected/actual observations in bin i.
    Bins are defined by the quantiles of the expected array.
    """
    # 1) define breakpoints by expected quantiles
    qs = np.linspace(0, 100, buckets + 1)
    breakpoints = np.percentile(expected, qs)

    # guard against duplicate bin edges
    breakpoints[0]  -= 1e-8
    breakpoints[-1] += 1e-8

    # 2) histogram counts
    exp_counts, _ = np.histogram(expected, bins=breakpoints)
    act_counts, _ = np.histogram(actual,   bins=breakpoints)

    # 3) convert to proportions
    exp_perc = exp_counts / exp_counts.sum()
    act_perc = act_counts / act_counts.sum()

    # 4) avoid zeros
    eps = 1e-8
    exp_perc = np.where(exp_perc == 0, eps, exp_perc)
    act_perc = np.where(act_perc == 0, eps, act_perc)

    # 5) compute PSI per bin and sum
    psi_vals = (act_perc - exp_perc) * np.log(act_perc / exp_perc)
    return psi_vals.sum()

# ——— now your main loop ———

results = pd.DataFrame(index=[dataset], columns=segments, dtype=float)

for s in segments:
    model_segment_filter = segment_map[s]
    # actual scores from prod (auto-approved rows only)
    dff = df[df['LOANPURPOSECATEGORY'] == model_segment_filter]
    dff = dff.groupby('APPID')['SCORE'].min()
    actual_df = pd.DataFrame(dff, columns=['SCORE'])

    # expected scores from dev
    exp_df = nonprod_scores[s].copy()
    exp_df.columns = ['SCORE']

    # align on the same APPIDs (or ZEST_KEYS → APPID, whichever index you used)
    idx = set(exp_df.index).intersection(actual_df.index)
    exp_vals = exp_df.loc[idx, 'SCORE'].values
    act_vals = actual_df.loc[idx, 'SCORE'].values

    # calculate PSI
    psi_value = calculate_psi(exp_vals, act_vals, buckets=10)
    results.loc[dataset, s] = psi_value

results
