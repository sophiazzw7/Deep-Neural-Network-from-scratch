import numpy as np
import pandas as pd

# ─── manual PSI implementation ────────────────────────────────────────────────
class PSI:
    def __init__(self, buckets: int = 10):
        self.buckets = buckets
        self.exp_df: pd.DataFrame = None
        self.breaks: dict[str, np.ndarray] = {}

    def _ensure_df(self, data):
        """Convert a Series-of-Series (or list‐of‐lists) into DataFrame, or pass through."""
        if isinstance(data, pd.Series):
            try:
                return data.apply(pd.Series)
            except Exception:
                raise ValueError(f"Cannot convert Series to DataFrame (dtype={data.dtype})")
        if not isinstance(data, pd.DataFrame):
            raise TypeError(f"Expected DataFrame or Series, got {type(data).__name__}")
        return data

    def fit(self, expected):
        """Learn bucket edges from non-prod (expected) data."""
        df = self._ensure_df(expected).copy()
        if df.empty:
            raise ValueError("Expected DataFrame is empty")
        self.exp_df = df
        for col in df.columns:
            vals = df[col].dropna().values
            if len(vals) == 0:
                raise ValueError(f"No non-null values in column '{col}'")
            qs = np.linspace(0, 1, self.buckets + 1)
            # unique to guard against constant columns
            self.breaks[col] = np.unique(np.quantile(vals, qs))

    def transform(self, actual) -> pd.Series:
        """Compute PSI for each feature vs. the fitted non-prod data."""
        df_act = self._ensure_df(actual)
        if self.exp_df is None:
            raise RuntimeError("`fit()` must be called before `transform()`")
        psi_vals = {}
        for col in df_act.columns:
            if col not in self.exp_df.columns:
                raise KeyError(f"Column '{col}' not seen in `fit()`")
            exp = self.exp_df[col].dropna().values
            act = df_act[col].dropna().values
            bins = self.breaks[col]

            exp_pct = np.histogram(exp, bins=bins)[0] / len(exp)
            act_pct = np.histogram(act, bins=bins)[0] / len(act)

            # floor zeros
            exp_pct = np.where(exp_pct == 0, 1e-8, exp_pct)
            act_pct = np.where(act_pct == 0, 1e-8, act_pct)

            psi_vals[col] = np.sum((act_pct - exp_pct) * np.log(act_pct / exp_pct))

        return pd.Series(psi_vals, name="psi")
# ────────────────────────────────────────────────────────────────────────────────

# ─── your existing setup ───────────────────────────────────────────────────────
segments    = ['auto','credit_card_consolidation','home_improvement','others']
segment_map = {
    'auto': 'Auto', 
    'credit_card_consolidation': 'Debt Consolidation', 
    'home_improvement': 'Home Improvement',
    'others': 'Others'
}
dataset    = 'test'
start_date = '2024-01-01'
end_date   = '2024-06-30'

# assume these come from your existing imports / earlier cells
nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(
    segments, dataset=dataset, subset='all'
)
fi_dict = get_nonprod_fi(segments)

df = get_prod_data_updated(
    start_date, end_date, subset='all'
)

# prepare results container
results = pd.DataFrame(
    np.zeros((1, len(segments))),
    index=[dataset],
    columns=segments
)

# ─── loop over segments ────────────────────────────────────────────────────────
for s in segments:
    model_segment_filter = segment_map[s]
    dffilt = df[df['LOANPURPOSECATEGORY'] == model_segment_filter]

    # both of these may be Series-of-Series
    prod_fe_data   = dffilt['DATA'].apply(pd.Series)
    nonprod_fe_df  = nonprod_fe[s]
    fi_df          = fi_dict[s]

    # compute PSI
    psi = PSI()
    psi.fit(nonprod_fe_df.fillna(-1))        # mirror your old fillna(-1)
    out = psi.transform(prod_fe_data)

    # pivot into a DataFrame with feature → psi
    psi_df = out.to_frame()
    psi_df.index.rename('feature', inplace=True)
    psi_df = psi_df.reset_index()            # now columns are ['feature','psi']

    # merge with importance and compute weighted average
    merged = psi_df.merge(fi_df, on='feature')
    wpsi   = (merged['psi'] * merged['importance']).sum() / merged['importance'].sum()

    results.loc[dataset, s] = wpsi

# ─── show your final table ─────────────────────────────────────────────────────
print(results.T)
