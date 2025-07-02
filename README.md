import numpy as np
import pandas as pd

def calculate_psi(expected_df: pd.DataFrame,
                  actual_df:   pd.DataFrame,
                  bins: int = 10) -> float:
    """
    Compute PSI between two DATAFrames that each have a 'SCORE' column.
    Bins are defined by quantiles of expected_df['SCORE'].
    Returns NaN if either input is empty.
    """
    exp = expected_df['SCORE'].values
    act = actual_df  ['SCORE'].values
    if exp.size == 0 or act.size == 0:
        return np.nan

    # 1) bin edges by expected quantiles
    qs = np.linspace(0, 100, bins + 1)
    edges = np.percentile(exp, qs)
    edges[0]  -= 1e-8
    edges[-1] += 1e-8

    # 2) histogram counts
    exp_cnt, _ = np.histogram(exp, bins=edges)
    act_cnt, _ = np.histogram(act, bins=edges)

    # 3) to proportions
    exp_pct = exp_cnt / exp_cnt.sum()
    act_pct = act_cnt / act_cnt.sum()

    # 4) avoid zeros for log
    eps = 1e-8
    exp_pct = np.where(exp_pct == 0, eps, exp_pct)
    act_pct = np.where(act_pct == 0, eps, act_pct)

    # 5) PSI formula
    return np.sum((act_pct - exp_pct) * np.log(act_pct / exp_pct))


# ——— now your main PSI loop ———
results = pd.DataFrame({s: np.zeros(1) for s in segments})
results.index = [dataset]

for s in segments:
    model_segment_filter = segment_map[s]

    # production-side
    prod_seg     = df[df['LOANPURPOSECATEGORY'] == model_segment_filter]
    prod_min     = prod_seg.groupby('APPID')['SCORE'].min()
    score_data   = pd.DataFrame(prod_min, columns=['SCORE'])

    # development-side
    nonprod_df   = nonprod_scores[s]
    nonprod_df.columns = ['SCORE']
    # filter to only the apps you auto-approved in dev
    nonprod_df = nonprod_df.loc[est_auto_approved_zest_keys.intersection(nonprod_df.index)]
    # bring in ZEST_KEY → APPID mapping
    nonprod_df = nonprod_df.merge(tar, left_index=True, right_index=True)
    nonprod_min = nonprod_df.groupby('applicationid')['SCORE'].min()
    nonprod_score_df = pd.DataFrame(nonprod_min, columns=['SCORE'])

    # compute PSI
    psi_val = calculate_psi(nonprod_score_df, score_data, bins=10)
    results.loc[dataset, s] = psi_val

# voila
print(results.T)
