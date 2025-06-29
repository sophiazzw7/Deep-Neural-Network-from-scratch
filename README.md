import itertools
import numpy as np
import pandas as pd
from sklearn.metrics import roc_auc_score

for score_type in score_ind:
    # 1) reproduce exactly the same filtering you do for "overall"
    dfagg = df[df[score_type] == df['BEST_' + score_type]]

    # 2) build one big DataFrame with (segment, score, target)
    frames = []
    for s in segments:
        cat = segment_map[s]                  # e.g. 'Auto', 'Debt Consolidation', etc.
        sub = dfagg[dfagg['LOANPURPOSECATEGORY'] == cat]

        sc = sub[[score_type]].rename(columns={score_type: 'score'})['score']
        tg = sub['target']

        keep = sc.notnull() & tg.notnull()
        sc = sc[keep].astype(float)
        tg = tg[keep].astype(int)

        # apply your PLRS flip if needed
        if score_type == 'PLRS':
            sc = (-1 * sc).astype(float)

        frames.append(
            pd.DataFrame({
                'segment': s,
                'score':   sc.values,
                'target':  tg.values
            })
        )

    big = pd.concat(frames, ignore_index=True)

    # 3) within-segment concordance
    within_conc = within_pairs = 0
    for g in segments:
        sub = big[big.segment == g]
        Np, Nn = (sub.target == 1).sum(), (sub.target == 0).sum()
        if Np * Nn == 0:
            continue
        auc_g = roc_auc_score(sub.target, sub.score)
        pairs = Np * Nn
        within_conc  += auc_g * pairs
        within_pairs += pairs

    # 4) cross-segment concordance
    cross_conc = cross_pairs = 0
    for g, h in itertools.permutations(segments, 2):
        pos = big[(big.segment == g) & (big.target == 1)].score.values
        neg = big[(big.segment == h) & (big.target == 0)].score.values
        Np, Nn = len(pos), len(neg)
        if Np * Nn == 0:
            continue
        y_true  = np.concatenate([np.ones(Np), np.zeros(Nn)])
        y_score = np.concatenate([pos, neg])
        auc_gh  = roc_auc_score(y_true, y_score)
        pairs   = Np * Nn
        cross_conc  += auc_gh * pairs
        cross_pairs += pairs

    # 5) overall AUC and print
    overall_auc = roc_auc_score(big.target, big.score)
    print(f"\n=== {score_type} ===")
    print(f"Overall AUC:           {overall_auc:.3f}")
    print(f"Within-segment AUC:   {within_conc/within_pairs:.3f}")
    print(f"Cross-segment AUC:    {cross_conc/cross_pairs:.3f}")
    total_pairs = within_pairs + cross_pairs
    print(f"Cross-segment weight: {cross_pairs/total_pairs:.1%}")
