from zaml.analyze.data_analysis.distribution_drift import PSI

segments    = ["auto", "credit_card_consolidation", "home_improvement", "others"]
segment_map = {
    "auto": "Auto",
    "credit_card_consolidation": "Debt Consolidation",
    "home_improvement": "Home Improvement",
    "others": "Others"
}
dataset     = "test_2"

# dev period
nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(
    segments, dataset=dataset, subset="all"
)
fi_dict     = get_nonprod_fi(segments)

# prod period (already filtered to auto‐approved ROWIDs)
df          = get_prod_data_updated(subset="all")

psi         = PSI()

results = pd.DataFrame({s: np.zeros(1) for s in segments})
results.index = [dataset]

for s in segments:
    model_segment_filter = segment_map[s]
    dffiltered = df[df["LOANPURPOSECATEGORY"] == model_segment_filter]

    # production‐side feature matrix
    prod_fe_data = dffiltered["DATA"].apply(pd.Series)

    # Shapley importances
    fi_df = fi_dict[s]

    # **COERCE** dev‐side features into a DataFrame
    nonprod_fe_df = nonprod_fe[s]
    # <— this line below is **all** you need to add 
    nonprod_fe_df = nonprod_fe_df.apply(pd.Series)

    # now run your existing black‐box PSI
    psi.fit(nonprod_fe_df)
    out = psi.transform(prod_fe_data)

    # turn into a DataFrame and merge
    psi_df = out.to_frame()
    psi_df.index = psi_df.index.rename("feature")
    psi_df = psi_df.rename({0: "psi"}, axis=1).reset_index()

    merged = psi_df.merge(fi_df, on="feature")
    # weighted average
    results.loc[dataset, s] = (
        (merged["psi"] * merged["importance"]).sum()
        / merged["importance"].sum()
    )

print(results.T)
