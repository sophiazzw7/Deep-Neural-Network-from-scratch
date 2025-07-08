Here is your revised version, grouped into the following sections: **Distribution of Variables**, **Outliers**, **Missing or Duplicate Values**, and **Consistency Checks**, while maintaining formal tone and structure throughout.

---

### 4.1 2024Q4 Data Quality Review

MRO independently reviewed all 273,639 application records across 44 columns to assess the integrity, completeness, and plausibility of the dataset. This evaluation was conducted as part of second-line model risk oversight and follows the same quality control standards applied in previous quarters.

#### Distribution of Variables

MRO assessed the range and distribution of key numerical fields. All model score values (SCORE) fell between 0 and 1, consistent with valid probability estimates. No negative values were observed in PLRS scores or any monetary variables. FICO scores ranged from 300 to 850, which conforms to standard consumer credit limits. These findings confirm that the dataset respects defined statistical and business constraints.

MRO also reviewed the distribution of the binary target variable. Of the total population, 267,756 records—or 97.85%—were labeled as non-defaults (TARGET = 0), while 5,883 records—or 2.15%—were classified as defaults (TARGET = 1). This is consistent with expectations for a prime-oriented unsecured installment loan portfolio.

In addition, MRO evaluated score calibration by PLRS decile. When accounts were grouped into ten equal-sized PLRS bands, the observed default rate declined steadily from approximately 4.3% in the lowest decile to 0.6% in the highest. This monotonic trend indicates that the PLRS score continues to rank-order credit risk appropriately.

#### Outliers

MRO investigated extreme values in the dataset. A total of 50 loans had funded amounts exceeding five standard deviations from the mean, with loan sizes ranging from \$125,000 to \$250,000. These values align with documented thresholds for high-ticket lending programs.

Similarly, 385 records exhibited unusually long approval-to-funding latencies, ranging from approximately 2.7 to 3.1 days—more than five standard deviations above the mean. Although these records lie on the tail of the latency distribution, they remain within operational tolerance and may reflect manual review workflows.

#### Missing or Duplicate Values

MRO verified that the dataset contained no duplicate application records when excluding the large JSON-style `DATA` column, confirming that each loan record is unique.

With respect to missing data, CHARGEOFFDATE and CHARGEOFFAMOUNT fields were absent in 268,799 records, which is approximately 98.2% of the file. This is expected, as only loans that have charged off populate these fields.

All eight delinquency count variables were missing in 437 records, or 0.16% of the dataset. These records had an observed default rate of 0%, suggesting that the missingness reflects the absence of any past-due events. MRO also identified a distinct segment of 437 accounts where all delinquency counters were absent, further supporting this interpretation.

Three legacy identifier fields—LIDS, DECISIONLIDS, and BEST\_LIDS—were missing in 108,050 records, representing 39.5% of the dataset. These variables are not model inputs, and the observed pattern is consistent with data exclusion practices for refinance transactions.

A total of four records were missing ZESTZAMLPROCESSINGID, and two were missing either PLRS or BEST\_PLRS scores. These data gaps represent less than 0.01% of the dataset and were assessed to be immaterial.

MRO also noted that 1,043 defaulted accounts lacked a recorded CHARGEOFFAMOUNT. These cases represent approximately 18% of all defaults and reflect known lags in data transmission between servicing systems and downstream reporting structures.

#### Consistency Checks

MRO evaluated the internal consistency of timestamp fields. In 17,702 records, or 6.5% of the dataset, the ApprovalDate was recorded as occurring after the FundedDate. Further inspection showed that nearly all of these anomalies took place within the same calendar day, likely due to timestamp formatting or sequencing.

In 260,675 records—accounting for 95.3% of the total—the FundedDate followed the Score\_Date, aligning with the intended application workflow. No records were found where ChargeoffDate preceded FundedDate, confirming proper temporal sequencing.

---

MRO's independent review of the 2024Q4 dataset revealed no material data quality issues. The observed patterns of missingness, value ranges, outlier behavior, and date relationships are consistent with business expectations and prior quarters. Based on these assessments, the dataset is deemed appropriate for model monitoring and validation.
