4.1 2024Q2 Data Quality Review

MRO independently reviewed all 214159 application records across 40 columns to assess the integrity, completeness, and plausibility of the dataset. This assessment was conducted as part of routine second-line oversight.

Distribution of Variables

MRO evaluated the distribution and range of key input variables. All model score values (SCORE) fell within the valid probability range of 0.0 to 1.0. No negative values were observed in PLRS scores or in any monetary amount fields. FICO scores ranged between 300 and 850, consistent with the conventional consumer credit scale. These findings confirm that all input features respect their natural business limits.

MRO reviewed the target distribution and found that 210351 records, or 98.2% of the total dataset, were labeled as non-defaults (TARGET = 0). The remaining 3808 records, or 1.8%, were identified as defaults (TARGET = 1). This level of class imbalance is consistent with expectations for a prime-focused unsecured installment loan portfolio.

An analysis of PLRS deciles revealed a monotonic trend: observed default rates declined steadily from approximately 3.6% in the lowest decile to 0.5% in the highest. This indicates that PD scores remained directionally aligned with credit risk across the spectrum.

Outliers

MRO investigated extreme values in funded amount and application latency. A total of 43 loans exhibited funded amounts greater than five standard deviations above the mean, with loan sizes between 125000 and 250000 dollars. These values fall within the plausible bounds of the portfolio's high-ticket loan offering.

In terms of latency between approval and funding, 255 records exceeded five standard deviations, with durations ranging from approximately 2.6 to 2.9 days. The median latency was 2 days, while the average was around 7 days. These extended durations are atypical but fall within operational limits.

Missing or Duplicate Values

MRO confirmed that the dataset contained no duplicated application records when excluding the large JSON-style DATA column. This verifies that each record is unique.

The CHARGEOFFDATE and CHARGEOFFAMOUNT fields were missing in 211237 records, or 98.7% of the dataset. These fields are populated only for charged-off loans, which comprise approximately 1.8% of the dataset.

All eight delinquency count features (COUNT60DPDWITHIN..., COUNT90DPDWITHIN...) were missing in 183 records, accounting for approximately 0.09% of the data. These accounts had a 0% observed default rate, suggesting that the absence of values corresponds to a lack of past-due history. MRO separately identified a group of 183 records with all DPD features missing.

Four records were missing ZESTZAMLPROCESSINGID, and one record was missing a PLRS score. These data gaps affected fewer than 0.01% of records and are not material.

Consistency Checks

MRO evaluated the temporal consistency of date-related fields. In 14011 records, or 6.5% of the dataset, the ApprovalDate appeared later than the FundedDate. Review of the timestamps revealed that most occurred within the same day, likely due to time-of-day sequencing.

In 203017 records, or 94.8%, the FundedDate followed the Score_Date as expected by the application process.

No instances were found where the ChargeoffDate preceded the FundedDate, confirming that the charge-off sequencing is logically valid.

MRO's independent data quality review for the 2024Q2 dataset did not reveal any material issues. All observed data patterns, missingness levels, value ranges, and timestamp relationships are consistent with prior expectations. Based on this review, the data is suitable for model monitoring and validation activities.
