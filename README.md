import numpy as np
import pandas as pd
from itertools import combinations
from sklearn.metrics import roc_auc_score

def weighted_within_auc(df, target='target', score='score', segment='segment'):
    """Weighted average of segment AUCs with weights = #pos·#neg pairs / total pairs."""
    total_pos = (df[target] == 1).sum()
    total_neg = (df[target] == 0).sum()
    total_pairs = total_pos * total_neg
    w_auc = 0.0
    for g, d in df.groupby(segment):
        n_pos = (d[target] == 1).sum()
        n_neg = (d[target] == 0).sum()
        if n_pos == 0 or n_neg == 0:
            continue                       # no pairs ⇒ no contribution
        w = (n_pos * n_neg) / total_pairs
        w_auc += w * roc_auc_score(d[target], d[score])
    return w_auc

overall_auc  = roc_auc_score(df['target'], df['score'])
within_auc   = weighted_within_auc(df)
cross_lift   = overall_auc - within_auc   # extra boost from cross-segment ranking

print(f"Overall  AUC : {overall_auc: .4f}")
print(f"Within   AUC : {within_auc : .4f}")
print(f"Cross-segment lift: {cross_lift : .4f}")
