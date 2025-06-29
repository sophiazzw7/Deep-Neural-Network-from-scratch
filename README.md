import itertools
import numpy as np
import pandas as pd
# make sure your existing auc() is in scope:
# from zaml.analyze.scores_analysis import ModelAUC
# def auc(scores, target): ...

for score_type in score_ind:
    # — reproduce your “overall” filtering —
    dfagg = df[df[score_type] == df['BEST_' + score_type]]

    # accumulators
    within_conc = within_pairs = 0
    cross_conc  = cross_pairs  = 0

    # 1) Within-segment
    for seg in segments:
        cat = segment_map[seg]
        sub = dfagg[dfagg['LOANPURPOSECATEGORY'] == cat]

        # build tiny DataFrames exactly like your main loop does
        sc_df = sub[[score_type]].rename(columns={score_type: 'final_model_predictions'})
        tg_df = sub[['target']].rename(columns={'target': 'final_ever60_18'})

        keep = sc_df['final_model_predictions'].notnull() & tg_df['final_ever60_18'].notnull()
        sc_df, tg_df = sc_df[keep], tg_df[keep]

        # flip PLRS if you do that in your loop
        if score_type == 'PLRS':
            sc_df['final_model_predictions'] *= -1

        Np = (tg_df['final_ever60_18'] == 1).sum()
        Nn = (tg_df['final_ever60_18'] == 0).sum()
        if Np * Nn == 0:
            continue

        auc_g = auc(sc_df, tg_df)
        pairs = Np * Nn
        within_conc  += auc_g * pairs
        within_pairs += pairs

    # 2) Cross-segment
    for seg_g, seg_h in itertools.permutations(segments, 2):
        cat_g, cat_h = segment_map[seg_g], segment_map[seg_h]
        pos = dfagg[(dfagg['LOANPURPOSECATEGORY'] == cat_g) & (dfagg['target'] == 1)]
        neg = dfagg[(dfagg['LOANPURPOSECATEGORY'] == cat_h) & (dfagg['target'] == 0)]

        pos_df = pos[[score_type]].rename(columns={score_type: 'final_model_predictions'})
        neg_df = neg[[score_type]].rename(columns={score_type: 'final_model_predictions'})

        # drop nulls
        pos_df = pos_df[pos_df['final_model_predictions'].notnull()]
        neg_df = neg_df[neg_df['final_model_predictions'].notnull()]

        Np, Nn = len(pos_df), len(neg_df)
        if Np * Nn == 0:
            continue

        # build combined tables
        sc_comb = pd.concat([pos_df, neg_df], ignore_index=True)
        tg_comb = pd.DataFrame({
            'final_ever60_18': [1]*Np + [0]*Nn
        })

        if score_type == 'PLRS':
            sc_comb['final_model_predictions'] *= -1

        auc_gh = auc(sc_comb, tg_comb)
        pairs = Np * Nn
        cross_conc  += auc_gh * pairs
        cross_pairs += pairs

    # 3) Overall (as your notebook does)
    overall_auc = auc(
        dfagg[[score_type]].rename(columns={score_type: 'final_model_predictions'}),
        dfagg[['target']].rename(columns={'target': 'final_ever60_18'})
    )

    # 4) Print a small summary
    within_auc = within_conc  / within_pairs
    cross_auc  = cross_conc   / cross_pairs
    total      = within_pairs + cross_pairs

    print(f"\n>>> {score_type} <<<")
    print(f" Overall AUC:           {overall_auc:.3f}")
    print(f" Within-segment AUC:   {within_auc:.3f}")
    print(f" Cross-segment AUC:    {cross_auc:.3f}")
    print(f" Cross-segment weight: {cross_pairs/total:.1%}")
