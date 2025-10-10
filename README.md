Hi [MD's Name],

I wanted to confirm a few details regarding the datasets used for the 202412 OGM review:

From my understanding, the three datasets — f_app_score_flat, intime_sample_30jun2015, and bdfs_am_final_summary_202412 — represent the full delivered output of the ETL run and serve as inputs to downstream reporting processes such as discrimination. Please let me know if that’s incorrect.

The reporting code also references another dataset, bdfs_local_fit. Could you please share this dataset as well?

Based on my frequency check, the vintage_yr distribution is heavily concentrated in 2023, with only a small portion from 2022 and the INTIME sample from 2015 (see attached). Does this pattern align with your expectations?

I also identified some duplicate (company_id, start_date, lob_indicator, sample_tp) combinations that carry different scores (attached). Could you confirm whether these are expected or if I should check with the developer?

Lastly, if my review is only intended for the 202412 OGM performance testing, should I subset the dataset using vintage_yr = 2023 to align with that reporting period?

Thank you for your time and guidance.

Best regards,
Phoebe
