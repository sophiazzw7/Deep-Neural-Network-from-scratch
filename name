import itertools
import numpy as np
import pandas as pd

for score_type in score_ind:
    dfagg = df[df[score_type] == df['BEST_' + score_type]]
    within_conc = within_pairs = cross_conc = cross_pairs = 0

    for seg in segments:
        cat = segment_map[seg]
        sub = dfagg[dfagg['LOANPURPOSECATEGORY'] == cat]
        sc_df = sub[[score_type]].rename(columns={score_type: 'final_model_predictions'})
        tg_df = sub[['target']].rename(columns={'target': 'final_ever60_18'})
        keep = sc_df['final_model_predictions'].notnull() & tg_df['final_ever60_18'].notnull()
        sc_df, tg_df = sc_df[keep], tg_df[keep]
        if score_type == 'PLRS':
            sc_df['final_model_predictions'] *= -1
        Np = (tg_df['final_ever60_18'] == 1).sum()
        Nn = (tg_df['final_ever60_18'] == 0).sum()
        if Np * Nn:
            auc_g = auc(sc_df, tg_df)
            pairs = Np * Nn
            within_conc  += auc_g * pairs
            within_pairs += pairs

    for seg_g, seg_h in itertools.permutations(segments, 2):
        cat_g = segment_map[seg_g]
        cat_h = segment_map[seg_h]
        pos = dfagg[(dfagg['LOANPURPOSECATEGORY'] == cat_g) & (dfagg['target'] == 1)]
        neg = dfagg[(dfagg['LOANPURPOSECATEGORY'] == cat_h) & (dfagg['target'] == 0)]
        pos_df = pos[[score_type]].rename(columns={score_type: 'final_model_predictions'})
        neg_df = neg[[score_type]].rename(columns={score_type: 'final_model_predictions'})
        pos_df = pos_df[pos_df['final_model_predictions'].notnull()]
        neg_df = neg_df[neg_df['final_model_predictions'].notnull()]
        Np = len(pos_df)
        Nn = len(neg_df)
        if Np * Nn:
            sc_comb = pd.concat([pos_df, neg_df], ignore_index=True)
            tg_comb = pd.DataFrame({'final_ever60_18': [1] * Np + [0] * Nn})
            if score_type == 'PLRS':
                sc_comb['final_model_predictions'] *= -1
            auc_gh = auc(sc_comb, tg_comb)
            pairs = Np * Nn
            cross_conc  += auc_gh * pairs
            cross_pairs += pairs

    overall_auc = auc(
        dfagg[[score_type]].rename(columns={score_type: 'final_model_predictions'}),
        dfagg[['target']].rename(columns={'target': 'final_ever60_18'})
    )
    within_auc = within_conc / within_pairs
    cross_auc  = cross_conc  / cross_pairs
    total_pairs = within_pairs + cross_pairs

    print(f"{score_type} overall AUC: {overall_auc:.3f}")
    print(f"within AUC:          {within_auc:.3f}")
    print(f"cross AUC:           {cross_auc:.3f}")
    print(f"cross weight:        {cross_pairs/total_pairs:.1%}")
