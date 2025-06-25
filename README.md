import pandas as pd

# 1) Defaults with missing charge‑off info
mask_default = df['TARGET'] == 1
missing_chargeoff = df[mask_default & df['CHARGEOFFAMOUNT'].isna()]
print(f"Defaults missing CHARGEOFFAMOUNT: {len(missing_chargeoff)} rows")
display(missing_chargeoff.head())

# 2) The 183 rows with no delinquency history
#    (one per COUNT*DPD* column, they all should agree)
delinq_cols = [c for c in df.columns if c.startswith('COUNT') and 'DPD' in c]
# find rows where *all* delinquency counts are null
mask_no_dpd = df[delinq_cols].isna().all(axis=1)
print("Rows with every COUNT*DPDWITHIN*MFUNDED missing:", mask_no_dpd.sum())
display(df.loc[mask_no_dpd, ['APPID'] + delinq_cols].head())

# 3) Approval → Funded date reversals
mask_bad_dates = df['APPROVALDATE'] > df['FUNDEDDATE']
print(f"Approval after funding: {mask_bad_dates.sum()} rows")
display(df.loc[mask_bad_dates, ['APPID','APPROVALDATE','FUNDEDDATE']].sample(10))

# 4) Records where you scored after funding (usually a logic check)
mask_score_after = df['SCORE_DATE'] > df['FUNDEDDATE']
print(f"Score after funding: {mask_score_after.sum()} rows")
display(df.loc[mask_score_after, ['APPID','SCORE_DATE','FUNDEDDATE']].sample(10))

# 5) Missingness vs. target
#    Do the 183 “no-delinquency” rows have a different default rate?
df['NO_DPD_HISTORY'] = mask_no_dpd
print("Default rate among NO_DPD_HISTORY:")
print(df.groupby('NO_DPD_HISTORY')['TARGET'].mean())

# 6) Field‐to‐field consistency: PLRS vs. BEST_PLRS when only one decision was made
mask_one_decision = df['DECISIONZORS'] == 1
bad_plrs = df[mask_one_decision & (df['PLRS'] != df['BEST_PLRS'])]
print("Single‐decision rows where PLRS≠BEST_PLRS:", len(bad_plrs))
display(bad_plrs.head())

# 7) Basic outlier hunt on FUNDEDAMOUNT and LATENCY
for col in ['FUNDEDAMOUNT','LATENCY']:
    mu, sigma = df[col].mean(), df[col].std()
    mask_out = (df[col] < mu - 5*sigma) | (df[col] > mu + 5*sigma)
    print(f"{col}: {mask_out.sum()} extreme (>±5σ) values")
    display(df.loc[mask_out, ['APPID',col]].head())
