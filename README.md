import pandas as pd
import numpy as np
from sklearn.metrics import roc_auc_score

# -----------------------------------------------------------------------------
# helper functions (unchanged)
# -----------------------------------------------------------------------------
def segment_stats(df_period):
    rows = []
    for seg, g in df_period.groupby("LOANPURPOSECATEGORY"):
        # need at least one 0 and one 1 to compute AUC
        if g["TARGET"].nunique() < 2:
            auc = np.nan
        else:
            auc = roc_auc_score(g["TARGET"], g["SCORE"])
        rows.append({
            "SEGMENT": seg,
            "N": len(g),
            "BadRate": g["TARGET"].mean(),
            "AUC": auc
        })
    out = pd.DataFrame(rows)
    out["Share"] = out["N"] / out["N"].sum()
    return out

def blended_auc(df_period):
    if df_period["TARGET"].nunique() < 2:
        return np.nan
    return roc_auc_score(df_period["TARGET"], df_period["SCORE"])

# -----------------------------------------------------------------------------
# 1) compute per-segment stats & overall AUC on each df
# -----------------------------------------------------------------------------
stats_q2 = segment_stats(df1)    # df1 holds 2024Q2
stats_q4 = segment_stats(df2)    # df2 holds 2024Q4

auc_q2 = blended_auc(df1)
auc_q4 = blended_auc(df2)

print(f"Overall blended AUC 2024Q2: {auc_q2:.3f}")
print(f"Overall blended AUC 2024Q4: {auc_q4:.3f}")

# -----------------------------------------------------------------------------
# 2) mix-shift decomposition (use Q2 shares × Q4 AUCs)
# -----------------------------------------------------------------------------
mix_adj_auc = (
    stats_q2.set_index("SEGMENT")["Share"]
     * stats_q4.set_index("SEGMENT")["AUC"]
).sum()

delta_total  = auc_q4 - auc_q2
delta_mix    = mix_adj_auc - auc_q2
delta_within = delta_total - delta_mix

print(f"\nMix-adjusted blended AUC (Q4 AUCs × Q2 volumes): {mix_adj_auc:.3f}")
print(f"Change in blended AUC: {delta_total:+.3f}")
print(f"  → from mix shift:      {delta_mix:+.3f}")
print(f"  → within-segment drift:{delta_within:+.3f}")

# -----------------------------------------------------------------------------
# 3) show per-segment breakdown side-by-side
# -----------------------------------------------------------------------------
combined = (
    stats_q2.assign(Period="2024Q2")
            .append(stats_q4.assign(Period="2024Q4"))
            .sort_values(["SEGMENT", "Period"])
)
print("\nPer-segment metrics:")
print(combined.to_string(index=False))
