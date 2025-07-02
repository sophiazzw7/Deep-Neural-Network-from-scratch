dev_raw = nonprod_fe[seg]
    if isinstance(dev_raw, pd.Series):
        dev_fe_df = dev_raw.apply(pd.Series)
    else:
        dev_fe_df = dev_raw.copy()
