from zaml.analyze.data_analysis.distribution_drift import PSI
import numpy as np
import pandas as pd

segments    = ["auto","credit_card_consolidation","home_improvement","others"]
segment_map = {
    "auto": "Auto",
    "credit_card_consolidation": "Debt Consolidation",
    "home_improvement": "Home Improvement",
    "others": "Others"
}
dataset     = "test_2"

nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(
    segments, dataset=dataset, subset="all"
)
fi_dict = get_nonprod_fi(segments)
df      = get_prod_data_updated(subset="all")

psi     = PSI()

results = pd.DataFrame({s: np.zeros(1) for s in segments})
results.index = [dataset]

for s in segments:
    model_segment_filter = segment_map[s]
    dffiltered = df[df["LOANPURPOSECATEGORY"] == model_segment_filter]

    # ——— PRODUCTION features ———
    raw_prod = dffiltered["DATA"]

    # 1) If .DATA is already a DataFrame, use it; otherwise explode the Series of dicts
    if isinstance(raw_prod, pd.DataFrame):
        prod_fe_data = raw_prod
    else:
        prod_fe_data = raw_prod.apply(pd.Series)

    fi_df = fi_dict[s]

    # ——— DEVELOPMENT features ———
    raw_dev = nonprod_fe[s]

    # 2) Same trick on the dev side
    if isinstance(raw_dev, pd.DataFrame):
        nonprod_fe_df = raw_dev
    else:
        nonprod_fe_df = raw_dev.apply(pd.Series)

    # Now both prod_fe_data and nonprod_fe_df are true DataFrames
    psi.fit(nonprod_fe_df)
    out = psi.transform(prod_fe_data)

    # convert the PSI output into a DataFrame exactly as before
    psi_df = out.to_frame()
    psi_df.index = psi_df.index.rename("feature")
    psi_df = psi_df.rename({0: "psi"}, axis=1).reset_index()

    merged = psi_df.merge(fi_df, on="feature")
    results.loc[dataset, s] = (
        (merged["psi"] * merged["importance"]).sum()
        / merged["importance"].sum()
    )

print(results.T)
