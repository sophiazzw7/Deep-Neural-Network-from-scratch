##############################################################################
# AUTO-APPROVED PSI  – feature + score
# • assumes you already ran the developer’s filter code that creates:
#     ─ est_auto_approved_zest_keys   (dev-side ZEST_KEY list)
#     ─ auto_approved_rowids          (prod-side RowID list)
# • assumes  get_nonprod_data / get_nonprod_fi / get_prod_data_updated
#   and start_date / end_date are in scope
##############################################################################

import numpy as np
import pandas as pd
from zaml.analyze.data_analysis.distribution_drift import PSI

# ── little helper to coerce PSI output into a tidy DataFrame ────────────────
def to_psi_df(raw, feature_index):
    """
    Return DF with columns ['feature','psi'] regardless of raw type.
    """
    if isinstance(raw, pd.DataFrame):      # already 2-D
        out = raw.copy()
        if out.shape[1] == 1:
            out.columns = ['psi']
        if 'feature' not in out.columns:
            out.insert(0, 'feature', feature_index)
        return out[['feature', 'psi']].reset_index(drop=True)

    if isinstance(raw, pd.Series):         # 1-D labelled
        return (raw.rename('psi')
                    .reset_index()
                    .rename(columns={'index': 'feature'}))

    # ndarray / list
    return pd.DataFrame({'feature': feature_index,
                         'psi':     np.asarray(raw).ravel()})


# ── 1.  segment labels ─────────────────────────────────────────────────────
segments = ['auto', 'credit_card_consolidation', 'home_improvement', 'others']
segment_map = {
    'auto': 'Auto',
    'credit_card_consolidation': 'Debt Consolidation',
    'home_improvement': 'Home Improvement',
    'others': 'Others',
}

# ── 2.  DEV data (restrict to auto-approved) ───────────────────────────────
nonprod_scores, nonprod_fe, _ = get_nonprod_data(
        segments, dataset='test_set', subset='all')
fi_dict = get_nonprod_fi(segments)

for seg in segments:
    nonprod_fe[seg] = (nonprod_fe[seg]
                       .loc[nonprod_fe[seg].index
                            .intersection(est_auto_approved_zest_keys)]
                       .fillna(-1))

# ── 3.  PROD data (restrict to auto-approved) ──────────────────────────────
prod_all = get_prod_data_updated(start_date, end_date, subset='approved')
prod_auto = prod_all[prod_all['RONID'].str.strip().isin(auto_approved_rowids)].copy()

# ── 4-A.  FEATURE-level importance-weighted PSI ────────────────────────────
feat_psi = pd.Series(index=segments, dtype=float)

for seg in segments:
    prod_seg = prod_auto[prod_auto['LOANPURPOSECATEGORY'] == segment_map[seg]]
    prod_feats = prod_seg['DATA'].apply(pd.Series).fillna(-1)

    psi = PSI()
    psi.fit(nonprod_fe[seg])
    drift_raw = psi.transform(prod_feats)
    psi_df = to_psi_df(drift_raw, nonprod_fe[seg].columns)

    merged = psi_df.merge(fi_dict[seg], on='feature')
    feat_psi[seg] = (merged['psi'] * merged['importance']).sum() / merged['importance'].sum()

print("\n=== Importance-weighted FEATURE PSI  (auto-approved only) ===")
display(feat_psi.to_frame('wPSI'))


# ── 4-B.  SCORE-level PSI (same logic the dev used) ────────────────────────
score_psi = pd.Series(index=segments, dtype=float)

for seg in segments:
    dev_score = (nonprod_scores[seg]
                 .rename(columns=lambda _: 'SCORE')
                 .loc[nonprod_scores[seg].index
                      .intersection(est_auto_approved_zest_keys)]
                 .fillna(-1))

    prod_score = (prod_auto[prod_auto['LOANPURPOSECATEGORY'] == segment_map[seg]]
                  .groupby('APPID')['SCORE'].min()
                  .to_frame('SCORE'))

    psi = PSI()
    psi.fit(dev_score)
    score_psi[seg] = to_psi_df(psi.transform(prod_score), ['score'])['psi'].iat[0]

print("\n=== MODEL-SCORE PSI  (auto-approved only) ===")
display(score_psi.to_frame('PSI'))
