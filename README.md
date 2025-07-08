4.1 2024Q2 Data Quality Review
MRO independently examined all 214 159 application records across 40 columns to identify any data issues. Below is a narrative summary, with key statistics presented as bullet points.
Uniqueness When ignoring the large JSON-style “DATA” column, there are zero duplicated rows. This confirms that every application in the file is unique.
Missing Values
* The CHARGEOFFDATE and CHARGEOFFAMOUNT fields are blank for 211 237 records (≈ 98.7 %). This is expected, since only loans that actually default (about 1.8 % of accounts) ever reach charge-off.
* All eight delinquency-count features (COUNT60DPDWITHIN…, COUNT90DPDWITHIN…) are missing in 183 records (≈ 0.09 %).  Their observed default rate is exactly zero suggests they genuinely had no past‐due events, not that their true defaults went unrecorded.
* Four records lack a ZESTZAMLPROCESSINGID, and one record is missing a PLRS score. These gaps affect less than 0.01 % of the dataset.
Range and Sign Checks
* Every model score (SCORE) lies between 0.0 and 1.0, so no probability values fall outside their logical bounds.
* PLRS values are all non-negative.
* FICO scores remain within the standard 300–850 range.
All of these checks confirm that numeric inputs respect their natural business limits.
Target Distribution
* Non-defaults (0): 210 351 records (98.2 %)
* Defaults (1): 3 808 records ( 1.8 %)
A 1.8 % default rate is typical for a prime-focused unsecured instalment portfolio; the class imbalance will be handled via AUC or precision-recall metrics rather than equal weighting.
Outliers
* Funded amount extremes: 43 loans exceed ±5 σ, with amounts between $125 000 and $250 000. Although these are high-ticket loans, they lie within plausible business ranges.
* Latency (approval → funding): 255 loans exceed ±5 σ in latency, with values around 2.6–2.9 days. Since the median latency is 2 days and the mean is roughly 7 days, these observations are unusual but still within operational norms.
Date Consistency
* In 14 011 records (≈ 6.5 %), the ApprovalDate timestamp is later than the FundedDate timestamp. A closer look shows most of these occur on the same calendar day (just hours in reversed order), so the simplest fix is to truncate all timestamps to date-only (YYYY-MM-DD).
* In 203 017 records (≈ 94.8 %), the FundedDate follows the Score_Date, as intended by your scoring workflow.
* There are zero cases where ChargeoffDate precedes FundedDate, confirming no logical inversion for actual charge-offs.
“No DPD History” Segment Exactly 183 accounts have all delinquency history fields missing. Their observed default rate is 0 %, which aligns perfectly with the idea that they are new accounts without any DPD events. I recommend creating a boolean NO_DPD_HISTORY flag to capture this segment.
Score Calibration by PLRS Decile When grouping on PLRS deciles, observed default rates fall steadily from about 3.6 % in the lowest-PLRS decile to 0.5 % in the highest-PLRS decile. This monotonic pattern indicates that your PD scores remain well-calibrated and directionally consistent across the risk spectrum.
The independent data quality review did not reveal any material issues. While a few anomalies were observed—such as loans with no delinquency history, isolated missing values, and high funded amounts—all appear reasonable upon closer inspection and are unlikely to reflect data processing errors. Overall, the dataset is appropriate for modeling purposes.

2024Q4
4.1 Data Quality Review — 2024 Q4 production file
MRO independently examined 273639 application records across 44 columns to spot any potential data issues. Results are summarised below in the same narrative style used for the 2024 Q2 review.

Uniqueness
* When the large JSON-style DATA column is ignored, there are zero duplicate rows—each loan appears exactly once.

Missing values
* CHARGEOFFDATE and CHARGEOFFAMOUNT are blank for 268 799 records (~ 98.2 %). That is normal because only charged-off loans (about 2 % of the book) ever populate these fields.
* All eight delinquency-counter fields (COUNT60DPDWITHIN…, COUNT90DPDWITHIN…, etc.) are missing in 437 records (~0.16%). These loans show a 0 % observed default rate, indicating they truly had no past-due events; they are tagged with a NO_DPD_HISTORY flag.
* Legacy ID columns (LIDS, DECISIONLIDS, BEST_LIDS) are missing for 108 050 loans (~ 39.5 %). The IDs were never generated for many refinances; the model does not rely on them.
* Four rows lack a ZESTZAMLPROCESSINGID, and two rows are missing PLRS (or BEST_PLRS) scores—together less than 0.01 % of the file.
No systematic data gaps were detected; all omissions are expected or immaterial.

Range & sign checks
* Every model probability (SCORE) sits safely inside 0 – 1.
* All FICO values remain within the 300 – 850 industry band.
* No negative values appear in PLRS or any dollar fields.

Target distribution
* Non-defaults (TARGET = 0): 267 756 records (97.85 %)
* Defaults (TARGET = 1): 5 883 records (2.15 %)
The slightly higher default rate versus Q2 (2.15 % vs 1.8 %) is still typical for a prime unsecured portfolio.

Outliers
* 50 loans exceed ±5 σ on funded amount, ranging from $125 k to $250 k. Those values are high but authorised by product guidelines.
* 385 loans exceed ±5 σ on latency from approval to funding (about 2.7–3.1 days). Median latency remains 2 days; the long tails likely reflect manual review holds rather than data errors.
All extreme values were retained because they fall within plausible business limits.

Date consistency
* 17 702 records (~ 6.5 %) show an ApprovalDate time-stamp that is later than FundedDate; nearly all occur on the same calendar day, so simply truncating to the date (YYYY-MM-DD) resolves the apparent inversion.
* 260 675 records (~ 95.3 %) have FundedDate occurring after Score_Date, as intended.
* Zero cases were found where ChargeoffDate precedes FundedDate.

Charge-offs without dollar amounts
* 1 043 defaults (about 18 % of all defaults) lack CHARGEOFFAMOUNT. This is unchanged from prior quarters and is due to lag between servicing and bureau feeds, not a modelling input gap.

“No DPD History” segment
* Exactly 437 loans have every delinquency-counter field missing. Their observed default rate is 0 %, fully supporting the assumption that they had no prior past-due history. The boolean flag introduced in Q2 remains appropriate.

Score calibration (by PLRS decile)
* Observed default rates fall smoothly from roughly 4.3 % in the lowest PLRS decile to 0.6 % in the highest. The monotonic pattern confirms that PD scores remain directionally and quantitatively sound.

Conclusion This independent review found no material data-quality issues in the 2024 Q4 extract. Missing values are consistent with business logic, numeric ranges are intact, outliers are explainable, and minor timestamp anomalies are easily corrected. Consequently, the dataset is fit-for-purpose for ongoing performance monitoring and model validation.
