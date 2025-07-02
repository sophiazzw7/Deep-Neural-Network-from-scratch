##############################################################################
#  AUTO-APPROVED PSI  (feature-level + score-level)
##############################################################################
import numpy as np
import pandas as pd
from zaml.analyze.data_analysis.distribution_drift import PSI

# ── helpers ─────────────────────────────────────────────────────────────────
def safe_df(obj, colnames=None):
    """
    Force obj into a 2-D pandas DataFrame.
    """
    if isinstance(obj, pd.DataFrame):
        out = obj.copy()
    elif isinstance(obj, pd.Series):
        out = obj.to_frame()
    else:                             # list / ndarray / scalar
        out = pd.DataFrame(obj)
    if colnames is not None:
        out.columns = colnames
    return out

def to_psi_df(raw, feature_index):
    """
    Return tidy DF  feature | psi  regardless of raw type.
    """
    if isinstance(raw, pd.DataFrame):
        if raw.shape[1] == 1:
            raw.columns = ['psi']
        if 'feature' not in raw.columns:
            raw.insert(0, 'feature', feature_index)
        return raw[['feature', 'psi']].reset_index(drop=True)

    if isinstance(raw, pd.Series):
        return raw.rename('psi').reset_index(names='feature')

    return pd.DataFrame({'feature': feature_index,
                         'psi':     np.asarray(raw).ravel()})

# ── 1.  segments & pretty names ─────────────────────────────────────────────
segments = ['auto', 'credit_card_consolidation', 'home_improvement', 'others']
segment_map = {
    'auto': 'Auto',
    'credit_card_consolidation': 'Debt Consolidation',
    'home_improvement': 'Home Improvement',
    'others': 'Others',
}

# ── 2.  DEV data  (restrict to auto-approved) ──────────────────────────────
nonprod_scores, nonprod_fe, _ = get_nonprod_data(
        segments, dataset='test_set', subset='all')
fi_dict = get_nonprod_fi(segments)

for seg in segments:
    nonprod_fe[seg] = (nonprod_fe[seg]
                       .loc[nonprod_fe[seg].index
                            .intersection(est_auto_approved_zest_keys)]
                       .fillna(-1))

# ── 3.  PROD data  (restrict to auto-approved) ─────────────────────────────
prod_all  = get_prod_data_updated(start_date, end_date, subset='approved')
prod_auto = prod_all[prod_all['RONID'].str.strip()
                     .isin(auto_approved_rowids)].copy()

# ── 4-A.  FEATURE-level importance-weighted PSI ────────────────────────────
feat_psi = pd.Series(index=segments, dtype=float)

for seg in segments:
    prod_seg = prod_auto[prod_auto['LOANPURPOSECATEGORY'] == segment_map[seg]]
    prod_feats = prod_seg['DATA'].apply(pd.Series)
    prod_feats = safe_df(prod_feats, colnames=nonprod_fe[seg].columns).fillna(-1)

    psi = PSI()
    psi.fit(safe_df(nonprod_fe[seg]))            # ensure 2-D
    drift_raw = psi.transform(prod_feats)
    psi_df = to_psi_df(drift_raw, nonprod_fe[seg].columns)

    merged = psi_df.merge(fi_dict[seg], on='feature')
    feat_psi[seg] = (merged['psi'] * merged['importance']).sum() / merged['importance'].sum()

print("\n=== Importance-weighted FEATURE PSI  (auto-approved only) ===")
display(feat_psi.to_frame('wPSI'))

# ── 4-B.  SCORE-level PSI (developer’s min-by-APPID logic) ─────────────────
score_psi = pd.Series(index=segments, dtype=float)

for seg in segments:
    dev_score = (nonprod_scores[seg].rename(columns=lambda _: 'SCORE')
                 .loc[nonprod_scores[seg].index
                      .intersection(est_auto_approved_zest_keys)]
                 .fillna(-1))
    dev_score = safe_df(dev_score, ['SCORE'])

    prod_score = (prod_auto[prod_auto['LOANPURPOSECATEGORY'] == segment_map[seg]]
                  .groupby('APPID')['SCORE'].min()
                  .to_frame('SCORE'))
    prod_score = safe_df(prod_score, ['SCORE'])

    psi = PSI()
    psi.fit(dev_score)
    score_psi[seg] = to_psi_df(psi.transform(prod_score), ['score'])['psi'].iat[0]

print("\n=== MODEL-SCORE PSI  (auto-approved only) ===")
display(score_psi.to_frame('PSI'))
