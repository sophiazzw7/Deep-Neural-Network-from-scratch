# … after you build & fit `psi` on dev_scores_dc …

# 1) Filter your production table for Debt Consolidation
mask = prod_all["LOANPURPOSE"] == "Debt Consolidation"
dc_scores_df = prod_all.loc[mask, ["SCORE"]]

# DEBUG: inspect
print("dev_scores_dc shape:", dev_scores_dc.shape)
print("dc_scores_df shape:  ", dc_scores_df.shape)
print("dc_scores_df head:\n", dc_scores_df.head())

# 2) Now call transform
psi_out = psi.transform(dc_scores_df)

# DEBUG: inspect output
print("psi_out type:", type(psi_out))
try:
    print("psi_out head:", psi_out.head())
except:
    print("psi_out repr:", psi_out)

# 3) Safely grab the first element if it exists
if len(psi_out) == 0:
    raise ValueError("PSI.transform returned no values – check that you fitted psi and that dc_scores_df is non-empty.")
else:
    psi_full = psi_out.iloc[0]
    print("Full-funnel Debt Conso PSI:", psi_full)
