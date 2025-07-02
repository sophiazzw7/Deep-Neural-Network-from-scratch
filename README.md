from zaml.analyze.data_analysis.distribution_drift import PSI

# 1) build dev_scores_dc correctly (many rows, one column "SCORE")
dev_raw = nonprod_scores['credit_card_consolidation']
dev_scores_dc = (
    dev_raw.to_frame.name=='SCORE' 
    if isinstance(dev_raw, pd.Series) 
    else pd.DataFrame({'SCORE': dev_raw})
)

# 2) fit PSI
psi = PSI()
psi.fit(dev_scores_dc)

# 3) filter prod_all and compute
mask = prod_all['LOANPURPOSE']=='Debt Consolidation'
prod_all_dc = prod_all.loc[mask, ['SCORE']]   # ensure this has >0 rows
print("Rows in DC:", len(prod_all_dc))

psi_full = psi.transform(prod_all_dc).iloc[0]
print("Full-funnel Debt Conso PSI:", psi_full)
