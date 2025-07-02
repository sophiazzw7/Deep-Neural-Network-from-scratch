# focus on the Debt Con segment under auto_only
seg = 'credit_card_consolidation'
feat_importance = fi_dict[seg]
# re-fit PSI on nonprod
psi = PSI()
psi.fit(nonprod_fe[seg].fillna(-1))

# extract prod_only auto-approved data
prod_seg_auto = (prod_df
                 [prod_df['LOANPURPOSECATEGORY'] == segment_map[seg]]
                 [prod_df['approvedByCurrentStrategies'] == True])
prod_feats_auto = prod_seg_auto['DATA'].apply(pd.Series)

# compute per-feature PSI
psi_vals = psi.transform(prod_feats_auto)
psi_df = (psi_vals
          .to_frame(name='psi')
          .rename_axis('feature')
          .reset_index())

# pick the top‐drift feature
top_feat = psi_df.sort_values('psi', ascending=False).iloc[0]['feature']
print("top PSI feature:", top_feat)

# pull raw values
dev_vals  = nonprod_fe[seg][top_feat].dropna()
prod_vals = prod_feats_auto[top_feat].dropna()

# plot normalized histograms
plt.figure()
plt.hist(dev_vals,  bins=30, density=True, alpha=0.6, label='Development')
plt.hist(prod_vals, bins=30, density=True, alpha=0.6, label='Production')
plt.xlabel(top_feat)
plt.ylabel('Normalized Frequency')
plt.title(f'Normalized Histogram Comparison for {top_feat}')
plt.legend()
plt.show()
