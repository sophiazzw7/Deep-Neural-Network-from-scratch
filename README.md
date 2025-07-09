Here’s a self-contained snippet you can drop into your notebook after you’ve loaded `df` (make sure it has columns `APPID`, `BEST_SCORE`, `target`, and `LOANPURPOSE`). It de-duplicates to one row per application (the max score), drops nulls, then computes AUC, KS and Gini by loan purpose and overall.

```python
import pandas as pd
import numpy as np
from sklearn.metrics import roc_curve, auc

# ------------------------------------------------------------------------------
# 1) Deduplicate: keep only the row with the highest BEST_SCORE per APPID
# ------------------------------------------------------------------------------
df_clean = df.copy()

# assume BEST_SCORE is the score column
df_clean = df_clean[df_clean['BEST_SCORE'] == df_clean.groupby('APPID')['BEST_SCORE'].transform('max')]

# ------------------------------------------------------------------------------
# 2) Drop any rows missing score or target
# ------------------------------------------------------------------------------
df_clean = df_clean.dropna(subset=['BEST_SCORE', 'target'])

# ------------------------------------------------------------------------------
# 3) Define metric functions
# ------------------------------------------------------------------------------
def compute_metrics(y_true, y_scores):
    # ROC / AUC
    fpr, tpr, _ = roc_curve(y_true, y_scores)
    roc_auc = auc(fpr, tpr)
    # KS = max distance between TPR and FPR
    ks_stat = np.max(np.abs(tpr - fpr))
    # Gini = 2 * AUC - 1
    gini = 2 * roc_auc - 1
    return roc_auc, ks_stat, gini

# ------------------------------------------------------------------------------
# 4) Loop over segments + overall
# ------------------------------------------------------------------------------
segments = df_clean['LOANPURPOSE'].unique().tolist()
results = []

for seg in segments + ['Overall']:
    if seg == 'Overall':
        sub = df_clean
    else:
        sub = df_clean[df_clean['LOANPURPOSE'] == seg]
    
    y_true   = sub['target'].astype(int)
    y_scores = sub['BEST_SCORE'].astype(float)
    
    roc_auc, ks_stat, gini = compute_metrics(y_true, y_scores)
    results.append({
        'Loan Purpose': seg,
        'AUC': roc_auc,
        'KS': ks_stat,
        'Gini': gini
    })

# ------------------------------------------------------------------------------
# 5) Display
# ------------------------------------------------------------------------------
results_df = pd.DataFrame(results)
print(results_df.to_string(index=False, float_format='%.4f'))
```

**What this does**

1. **Deduplication**: `groupby(APPID).transform('max')` ensures one score per loan.
2. **Null-filtering**: removes any apps without a valid score or target.
3. **Metrics**: uses `sklearn`’s ROC to get AUC, then KS as `max|TPR−FPR|`, and Gini = 2·AUC−1.
4. **Loop**: runs per segment and then “Overall.”

That should reproduce the developer’s ZORS numbers exactly.
