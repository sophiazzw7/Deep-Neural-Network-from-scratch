# … after you load prod_all …

# 1) Campaign window for Debt Consolidation
camp = prod_all[
    (prod_all["LOANPURPOSECATEGORY"] == "Debt Consolidation")
    & (pd.to_datetime(prod_all["SCORE_DATE"]) >= "2024-11-01")
    & (pd.to_datetime(prod_all["SCORE_DATE"]) <= "2024-12-31")
]

print("=== Campaign volumes & decision rates ===")
print(camp["DECISIONZORS"].value_counts())            # raw counts
print(camp["DECISIONZORS"].value_counts(normalize=True))  # percentages

# 2) Score-PSI check (two-sample KS) for this segment
from zaml.analyze.data_analysis.distribution_drift import PSI

psi = PSI()
# dev_scores_dc must be a DataFrame with one column "SCORE"
psi.fit(dev_scores_dc)

# Full funnel
psi_full = psi.transform(prod_all.loc[
    prod_all["LOANPURPOSECATEGORY"]=="Debt Consolidation", ["SCORE"]
])[0]
# Auto-approved subset
psi_auto = psi.transform(prod_auto.loc[
    prod_auto["LOANPURPOSECATEGORY"]=="Debt Consolidation", ["SCORE"]
])[0]

print(f"Full-funnel PSI = {psi_full:.3f}, Auto-approved PSI = {psi_auto:.3f}")
