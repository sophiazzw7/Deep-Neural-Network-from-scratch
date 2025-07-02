from zaml.analyze.data_analysis.distribution_drift import PSI
import pandas as pd
import numpy as np

# Your custom PSI function
def custom_psi(expected_df, actual_df, feature_col='feature', bins=10):
    """
    Custom PSI (Population Stability Index) calculation
    
    Parameters:
    expected_df: DataFrame with expected/baseline data
    actual_df: DataFrame with actual/comparison data  
    feature_col: column name containing the feature values
    bins: number of bins for discretization
    
    Returns:
    DataFrame with PSI results
    """
    
    # Example PSI calculation - replace with your logic
    def calculate_psi_score(expected, actual, bins=bins):
        # Discretize the data into bins
        try:
            # Create bins based on expected data quantiles
            bin_edges = pd.qcut(expected, q=bins, duplicates='drop', retbins=True)[1]
            
            # Apply same bins to both datasets
            expected_binned = pd.cut(expected, bins=bin_edges, include_lowest=True, duplicates='drop')
            actual_binned = pd.cut(actual, bins=bin_edges, include_lowest=True, duplicates='drop')
            
            # Calculate proportions
            expected_props = expected_binned.value_counts(normalize=True, sort=False)
            actual_props = actual_binned.value_counts(normalize=True, sort=False)
            
            # Align indices and fill missing with small value to avoid log(0)
            expected_props = expected_props.reindex(actual_props.index, fill_value=1e-10)
            actual_props = actual_props.fillna(1e-10)
            expected_props = expected_props.fillna(1e-10)
            
            # Calculate PSI
            psi_values = (actual_props - expected_props) * np.log(actual_props / expected_props)
            psi_score = psi_values.sum()
            
            return psi_score
            
        except Exception as e:
            print(f"Error calculating PSI: {e}")
            return np.nan
    
    # Calculate PSI score
    psi_score = calculate_psi_score(expected_df[feature_col], actual_df[feature_col])
    
    # Return in format similar to original PSI function
    result_df = pd.DataFrame({
        'feature': [feature_col],
        'psi': [psi_score]
    })
    
    return result_df

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
    
    # Use your custom PSI function instead of PSI()
    # Assuming dffiltered contains production features in same format as nonprod_fe_df
    prod_fe_data = dffiltered  # You may need to transform this to match nonprod_fe_df format
    
    # Apply custom PSI
    psi_results = custom_psi(nonprod_fe_df, prod_fe_data, feature_col='feature')  # Adjust column name as needed
    
    psi_df = psi_results
    psi_df.index = psi_df.index.rename('feature')
    psi_df = psi_df.rename({'psi'}, axis=1)
    psi_df = psi_df.reset_index()
    
    merged = psi_df.merge(fi_df, on="feature")
    results.loc[dataset, s] = (merged['psi'] * merged['importance']).sum() / merged['importance'].sum()

results.T
