import pandas as pd

# dev_raw might be a Series *or* a 1-col DataFrame
dev_raw = nonprod_scores[segment_key]

if isinstance(dev_raw, pd.Series):
    # Series → single‐column DF
    dev_df = dev_raw.to_frame(name="SCORE")

elif isinstance(dev_raw, pd.DataFrame):
    # already a DataFrame: just rename its sole column to “SCORE”
    if dev_raw.shape[1] != 1:
        raise ValueError("Expected nonprod_scores[...] to be 1-column, got %d cols" 
                         % dev_raw.shape[1])
    dev_df = dev_raw.copy()
    dev_df.columns = ["SCORE"]

else:
    # scalar or list/array
    dev_df = pd.DataFrame(dev_raw, columns=["SCORE"])

print("✔ dev_df is now an N×1 DataFrame with column ‘SCORE’")
print(dev_df.shape)
print(dev_df.head())
