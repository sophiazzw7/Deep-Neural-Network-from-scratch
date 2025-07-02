import numpy as np

def manual_psi_eqfreq(baseline, current, bins=10, floor=1e-6):
    """
    PSI using equal-frequency bins defined on 'baseline' only.
    
    baseline: 1d array of dev scores
    current:  1d array of prod scores (same metric)
    bins:     number of quantile bins (commonly 10 for deciles)
    floor:    small value to avoid log(0)
    """
    # 1) get the bin edges from the baseline quantiles
    quantiles = np.linspace(0, 100, bins + 1)
    edges = np.percentile(baseline, quantiles)
    
    # make sure edges are strictly increasing
    edges[0]  -= 1e-8    # include anything == min
    edges[-1] += 1e-8    # include anything == max
    
    # 2) histogram counts
    b_cnts, _ = np.histogram(baseline, bins=edges)
    c_cnts, _ = np.histogram(current,  bins=edges)
    
    # 3) convert to proportions + floor
    b_pct = b_cnts / (b_cnts.sum() + floor) + floor
    c_pct = c_cnts / (c_cnts.sum() + floor) + floor
    
    # 4) PSI sum
    return np.sum((c_pct - b_pct) * np.log(c_pct / b_pct))


# ── now recompute your auto-approved PSI using eq-freq bins ────────────────

dev_auto_arr  = dev_df_auto["SCORE"].values
prod_auto_arr = prod_df_auto["SCORE"].values

psi_auto_deciles = manual_psi_eqfreq(dev_auto_arr, prod_auto_arr, bins=10)
print(f"Auto-approved PSI (decile bins): {psi_auto_deciles:.3f}")
