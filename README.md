dev_raw = nonprod_scores['credit_card_consolidation']

# 1) Inspect what you got:
print(type(dev_raw), 
      "shape" if hasattr(dev_raw, 'shape') else "", 
      getattr(dev_raw, 'shape', dev_raw))

# 2) Turn it into a DataFrame of shape (n,1)
if isinstance(dev_raw, pd.Series):
    dev_scores_dc = dev_raw.to_frame(name='SCORE')
elif hasattr(dev_raw, '__len__'):  # e.g. a numpy array
    dev_scores_dc = pd.DataFrame(dev_raw, columns=['SCORE'])
else:
    # scalar: wrap it in a list and give it an index
    dev_scores_dc = pd.DataFrame({'SCORE':[dev_raw]}, index=[0])

print(dev_scores_dc.head())
