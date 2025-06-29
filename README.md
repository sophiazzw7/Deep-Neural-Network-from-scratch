import itertools
import numpy as np
import pandas as pd
from sklearn.metrics import roc_auc_score

for score_type in score_ind:
    # —— reproduce your overall‐AUC filtering ——
    dfagg = df[df[score_type] == df['BEST_' + score_type]]

    # —— build a combined DataFrame of (segment, score, target) ——
    frames = []
    for s in segments:
        cat = segment_map[s]
        sub = dfagg[dfagg['LOANPURPOSECATEGORY'] == cat]

        # grab & rename just like in your loop
        scores = sub[[score_type]].rename(columns={score_type: 'score'})
        target = sub[['target']].rename(columns={'target': 'target'})
        keep = scores['score'].notnull() & target['target'].notnull()

        # apply your PLRS sign‐flip if needed
        sc = scores.loc[keep, 'score']
        if score_type == 'PLRS':
            sc = (sc.astype(int) * -1).astype(float)
        else:
            sc = sc.astype(float)

        tg = target.loc[keep, 'target'].astype(int)

        frames.append(pd.DataFrame({
            'segment': s,
            'score':   sc.values,
            'target':  tg.values
        }))

    big = pd.concat(frames, ignore_index=True)

    # —— within‐segment concordance —— 
    within_concordant = within_pairs = 0
    for g in segments:
        sub = big[big.segment == g]
        Np = (sub.target == 1).sum()
        Nn = (sub.target == 0).sum()
        if Np * Nn == 0:
            continue
        auc_g = roc_auc_score(sub.target, sub.score)
        pairs = Np * Nn
        within_concordant += auc_g * pairs
        within_pairs     += pairs

    # —— cross‐segment concordance ——
    cross_concordant = cross_pairs = 0
    for g, h in itertools.permutations(segments, 2):
        pos = big[(big.segment == g) & (big.target == 1)].score.values
        neg = big[(big.segment == h) & (big.target == 0)].score.values
        Np, Nn = len(pos), len(neg)
        if Np * Nn == 0:
            continue
        y_true  = np.concatenate([np.ones(Np), np.zeros(Nn)])
        y_score = np.concatenate([pos, neg])
        auc_gh = roc_auc_score(y_true, y_score)
        pairs  = Np * Nn
        cross_concordant += auc_gh * pairs
        cross_pairs      += pairs

    # —— results —— 
    overall_auc = roc_auc_score(big.target, big.score)
    print(f"\n▶ {score_type} ▶ overall AUC: {overall_auc:.3f}")
    print(f"    within-segment AUC: {within_concordant/within_pairs:.3f}")
    print(f"    cross-segment AUC:  {cross_concordant/cross_pairs:.3f}")
    print(f"    cross-segment weight: {cross_pairs/(within_pairs+cross_pairs):.1%}")
