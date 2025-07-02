##############################################################################
# 0.  PREREQUISITES ––– run the developer’s cells first
#     -------------------------------------------------
#     •  auto_approved_appids            (set of APPIDs in test_2 current-strategy file)
#     •  est_auto_approved_zest_keys     (set of ZEST_KEYs mapped from those APPIDs)
#     •  auto_approved_rowids            (set of RowId’s flagged WasAutoApproved == 1
#                                         in the compiled-Zest production CSV)
##############################################################################

from zaml.analyze.data_analysis.distribution_drift import PSI
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

# 1.  segments & pretty names – unchanged
segments = ['auto', 'credit_card_consolidation', 'home_improvement', 'others']
segment_map = {
    'auto': 'Auto',
    'credit_card_consolidation': 'Debt Consolidation',
    'home_improvement': 'Home Improvement',
    'others': 'Others',
}

# 2.  pull DEV (non-prod) features / importance – unchanged
nonprod_scores, nonprod_fe, _ = get_nonprod_data(
        segments, dataset='test_set', subset='all')
fi_dict = get_nonprod_fi(segments)

# *** restrict DEV to auto-approved ***
for seg in segments:
    nonprod_fe[seg] = (
        nonprod_fe[seg]                         # keep same feature set
        .loc[nonprod_fe[seg].index
             .intersection(est_auto_approved_zest_keys)]
        .fillna(-1)
    )

# 3.  pull PROD → filter to auto-approved rowids once, then reuse
prod_df_full = get_prod_data_updated(start_date, end_date, subset='approved')
prod_df_auto = prod_df_full[
        prod_df_full['RONID'].str.strip().isin(auto_approved_rowids)].copy()

# 4.  run PSI per segment  (dev auto-approved  vs  prod auto-approved)
results = pd.Series(index=segments, dtype=float)

for seg in segments:
    df_prod_seg = prod_df_auto[
        prod_df_auto['LOANPURPOSECATEGORY'] == segment_map[seg]]

    # explode JSON/struct column → regular wide DF
    prod_feats = df_prod_seg['DATA'].apply(pd.Series).fillna(-1)

    psi = PSI()
    psi.fit(nonprod_fe[seg])          # dev   auto-approved
    drift = psi.transform(prod_feats) # prod  auto-approved

    # importance-weighted aggregation
    psi_df = (drift.to_frame('psi')
                    .reset_index()
                    .rename(columns={'index': 'feature'}))
    merged = psi_df.merge(fi_dict[seg], on='feature')
    wpsi = (merged['psi'] * merged['importance']).sum() / merged['importance'].sum()
    results[seg] = wpsi

print("=== importance-weighted feature PSI  (auto-approved only) ===")
display(results.to_frame('wPSI'))

##############################################################################
# 5.  OPTIONAL – score PSI (one value per segment) -- replicates dev’s cell
##############################################################################
score_results = pd.Series(index=segments, dtype=float)
for seg in segments:
    # DEV (restricted)
    nonprod_score_df = nonprod_scores[seg].rename(columns=lambda _: 'SCORE')
    nonprod_score_df = (
        nonprod_score_df
        .loc[nonprod_score_df.index.intersection(est_auto_approved_zest_keys)]
        .fillna(-1)
    )

    # PROD (restricted)
    score_prod = (
        prod_df_auto[prod_df_auto['LOANPURPOSECATEGORY'] == segment_map[seg]]
        .groupby('APPID')['SCORE'].min()     # same min() logic the dev used
        .to_frame('SCORE')
    )

    psi = PSI()
    psi.fit(nonprod_score_df)
    score_results[seg] = psi.transform(score_prod)[0]

print("\n=== model-score PSI  (auto-approved only) ===")
display(score_results.to_frame('PSI'))
