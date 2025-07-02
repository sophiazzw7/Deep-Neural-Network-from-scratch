import pandas as pd
import matplotlib.pyplot as plt
from zaml.analyze.data_analysis.distribution_drift import PSI
# (Replace these with your actual import paths)
from your_project.data_loader import get_nonprod_data, get_prod_data_updated
from your_project.model_inspect import get_feature_importance  

# 1) Load the “Debt Consolidation” development and production slices ----------
dev_scores, dev_feats, _ = get_nonprod_data(
    ['credit_card_consolidation'], dataset='test_2', subset='all'
)
dev_scores_dc = dev_scores['credit_card_consolidation'].to_frame('SCORE')
dev_feats_dc   = dev_feats['credit_card_consolidation']

prod_all = get_prod_data_updated('2024-07-01','2024-12-31', subset='all')
prod_auto = get_prod_data_updated('2024-07-01','2024-12-31', subset='auto_approved')

# filter to DC segment
mask_dc       = prod_all['LOANPURPOSECATEGORY']=='Debt Consolidation'
prod_all_dc   = prod_all[mask_dc]
prod_auto_dc  = prod_auto[prod_auto['LOANPURPOSECATEGORY']=='Debt Consolidation']

# 2) Check marketing-campaign volumes & rates ----------------------------------
# campaign window
camp = prod_all_dc[
    (prod_all_dc['APPLICATION_DATE']>='2024-11-01') &
    (prod_all_dc['APPLICATION_DATE']<='2024-12-31')
]

print("=== Campaign volumes & decision rates ===")
print(camp['DECISION_TYPE'].value_counts())
print(camp['DECISION_TYPE'].value_counts(normalize=True))

# 3) Compute PSI on scores (two-sample) ---------------------------------------
psi = PSI()
psi.fit(dev_scores_dc)

psi_full = psi.transform(prod_all_dc[['SCORE']])[0]
psi_auto = psi.transform(prod_auto_dc[['SCORE']])[0]
print(f"\nDebt Conso Score-PSI → Full funnel: {psi_full:.3f}, Auto-approved only: {psi_auto:.3f}")

# 4) Identify top feature by Shapley weight and compare distributions ----------
fi = get_feature_importance('credit_card_consolidation')  # DataFrame with ['feature','importance']
top_feat = fi.sort_values('importance', ascending=False).iloc[0,'feature']
print(f"\nTop feature for DC: {top_feat}")

# extract development vs production values
dev_f = dev_feats_dc[top_feat]
# assume prod_all_dc stores feature JSON in column 'DATA'
prod_f = pd.json_normalize(prod_all_dc['DATA'])[top_feat]

print(f"Dev mean {top_feat}: {dev_f.mean():.2f}")
print(f"Prod mean {top_feat}: {prod_f.mean():.2f}")

# histogram for visual
plt.figure(figsize=(8,4))
plt.hist(dev_f, bins=50, alpha=0.6, label='Dev',   density=True)
plt.hist(prod_f, bins=50, alpha=0.6, label='Prod',  density=True)
plt.title(f"{top_feat} — Dev vs. Prod")
plt.legend()
plt.show()
