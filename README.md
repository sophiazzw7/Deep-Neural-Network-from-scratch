# 1) See which column has the right labels
print(prod_all["LOANPURPOSE"].unique())

# 2) Filter on that column
mask = prod_all["LOANPURPOSE"] == "Debt Consolidation"
print("Rows in DC:", mask.sum())   # should be > 0

prod_all_dc = prod_all[mask]

# 3) Now compute PSI
psi_full = psi.transform(prod_all_dc[["SCORE"]])[0]
print("Full-funnel Debt Conso PSI:", psi_full)
