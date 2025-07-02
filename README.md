from zaml.analyze.data_analysis.distribution_drift import PSI  # This line can be removed
import pandas as pd
import numpy as np

# Standard PSI calculation function
def PSI(nonprod_fe_df):
    """
    Standard PSI (Population Stability Index) calculation
    Replaces the ZAML PSI function with standard implementation
    
    Parameters:
    nonprod_fe_df: DataFrame with feature data
    
    Returns:
    Object with transform method that returns PSI results
    """
    
    class PSICalculator:
        def __init__(self, baseline_df):
            self.baseline_df = baseline_df
            
        def transform(self, prod_fe_data):
            """
            Calculate PSI for each feature comparing baseline to production data
            """
            psi_results = []
            
            # Get feature columns (assuming all columns are features)
            feature_cols = self.baseline_df.columns
            
            for feature in feature_cols:
                if feature in prod_fe_data.columns:
                    try:
                        psi_score = self._calculate_psi_for_feature(
                            self.baseline_df[feature], 
                            prod_fe_data[feature]
                        )
                        psi_results.append({'feature': feature, 'psi': psi_score})
                    except Exception as e:
                        print(f"Error calculating PSI for {feature}: {e}")
                        psi_results.append({'feature': feature, 'psi': np.nan})
            
            return pd.DataFrame(psi_results)
        
        def _calculate_psi_for_feature(self, expected, actual, bins=10):
            """
            Standard PSI calculation for a single feature
            PSI = Σ (Actual% - Expected%) * ln(Actual% / Expected%)
            """
            # Handle missing values
            expected = expected.dropna()
            actual = actual.dropna()
            
            if len(expected) == 0 or len(actual) == 0:
                return np.nan
            
            # Determine if feature is categorical or numerical
            if expected.dtype == 'object' or expected.dtype.name == 'category':
                return self._calculate_psi_categorical(expected, actual)
            else:
                return self._calculate_psi_numerical(expected, actual, bins)
        
        def _calculate_psi_categorical(self, expected, actual):
            """PSI calculation for categorical features"""
            # Get proportions for expected (baseline)
            expected_props = expected.value_counts(normalize=True)
            actual_props = actual.value_counts(normalize=True)
            
            # Get all unique categories
            all_categories = set(expected_props.index) | set(actual_props.index)
            
            psi_sum = 0
            for category in all_categories:
                expected_pct = expected_props.get(category, 1e-10)  # Small value to avoid log(0)
                actual_pct = actual_props.get(category, 1e-10)
                
                if expected_pct > 0 and actual_pct > 0:
                    psi_sum += (actual_pct - expected_pct) * np.log(actual_pct / expected_pct)
            
            return psi_sum
        
        def _calculate_psi_numerical(self, expected, actual, bins=10):
            """PSI calculation for numerical features"""
            try:
                # Create bins based on expected data quantiles
                _, bin_edges = pd.qcut(expected, q=bins, duplicates='drop', retbins=True)
                
                # If we have fewer unique values than bins, use unique values
                if len(bin_edges) <= 2:
                    unique_vals = sorted(expected.unique())
                    if len(unique_vals) > 1:
                        bin_edges = unique_vals + [float('inf')]
                    else:
                        return 0.0  # No variation
                
                # Bin both datasets using the same edges
                expected_binned = pd.cut(expected, bins=bin_edges, include_lowest=True, duplicates='drop')
                actual_binned = pd.cut(actual, bins=bin_edges, include_lowest=True, duplicates='drop')
                
                # Calculate proportions
                expected_props = expected_binned.value_counts(normalize=True, sort=False)
                actual_props = actual_binned.value_counts(normalize=True, sort=False)
                
                # Ensure all bins are present in both
                all_bins = expected_props.index.union(actual_props.index)
                expected_props = expected_props.reindex(all_bins, fill_value=1e-10)
                actual_props = actual_props.reindex(all_bins, fill_value=1e-10)
                
                # Calculate PSI
                psi_values = (actual_props - expected_props) * np.log(actual_props / expected_props)
                return psi_values.sum()
                
            except Exception as e:
                # Fallback to simple binning
                return self._calculate_psi_simple_bins(expected, actual, bins)
        
        def _calculate_psi_simple_bins(self, expected, actual, bins=10):
            """Fallback PSI calculation with simple binning"""
            try:
                # Use histogram binning
                min_val = min(expected.min(), actual.min())
                max_val = max(expected.max(), actual.max())
                bin_edges = np.linspace(min_val, max_val, bins + 1)
                
                expected_hist, _ = np.histogram(expected, bins=bin_edges)
                actual_hist, _ = np.histogram(actual, bins=bin_edges)
                
                # Convert to proportions
                expected_props = expected_hist / expected_hist.sum()
                actual_props = actual_hist / actual_hist.sum()
                
                # Add small value to avoid log(0)
                expected_props = np.where(expected_props == 0, 1e-10, expected_props)
                actual_props = np.where(actual_props == 0, 1e-10, actual_props)
                
                # Calculate PSI
                psi = np.sum((actual_props - expected_props) * np.log(actual_props / expected_props))
                return psi
                
            except:
                return np.nan
    
    return PSICalculator(nonprod_fe_df)

# Your existing code structure with custom PSI
segments = ["auto", "credit_card_consolidation", "home_improvement", "others"]
segment_map = {
    'auto': 'Auto', 
    'credit_card_consolidation': 'Debt Consolidation', 
    'home_improvement': 'Home Improvement', 
    'others': 'Others'
}

dataset = 'test2'
nonprod_scores, nonprod_fe, nonprod_target = get_nonprod_data(segments, dataset=dataset, subset='all')
fi_dict = get_nonprod_fi(segments)
df = get_prod_data_updated(start_date, end_date, subset='all')

# Initialize results
results = pd.DataFrame({'s': np.zeros(1) for s in segments})
results.index = [dataset]

for s in segments:
    model_segment_filter = segment_map[s]
    dffiltered = df[df['LOANPURPOSECATEGORY'] == model_segment_filter]
    dffiltered['DATA'].apply(pd.Series)
    fi_df = fi_dict[s]
    nonprod_fe_df = nonprod_fe[s]
    
    # Fill missing values
    nonprod_fe_df = nonprod_fe_df.fillna(-1)
    
    # Extract production feature data from dffiltered to match nonprod_fe_df format
    # You may need to adjust this based on your data structure
    prod_fe_data = dffiltered[['DATA']].copy()  # Adjust column selection as needed
    
    # Use standard PSI calculation (replacing ZAML PSI)
    psi = PSI(nonprod_fe_df)
    out = psi.transform(prod_fe_data)
    psi_df = out
    psi_df.index = psi_df.index.rename('feature')
    psi_df = psi_df.rename({'psi'}, axis=1)
    psi_df = psi_df.reset_index()
    
    merged = psi_df.merge(fi_df, on="feature")
    results.loc[dataset, s] = (merged['psi'] * merged['importance']).sum() / merged['importance'].sum()

results.T
