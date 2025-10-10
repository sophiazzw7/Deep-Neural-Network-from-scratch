proc sql;
select count(*) as neg_scores
from bdfs_am_final_summary_202412
where sample_tp='CURRENT' and score_value < 100;
quit;



/* 🔍 Check if weight=15.478 exists in CURRENT data */
proc sql;
  select count(*) as n_weight_15478
  from bdfs_am_final_summary_202412
  where sample_tp = 'CURRENT' and abs(weight - 15.478) < 0.0001;
quit;

/* 🔍 If you want to see where they occur (by vintage or LOB) */
proc sql outobs=20;
  select vintage_yr, lob_indicator, score_value, bad_at_12mo, weight
  from bdfs_am_final_summary_202412
  where sample_tp='CURRENT' and abs(weight - 15.478) < 0.0001;
quit;

/* 🔍 Optionally check if this value shows up for any CURRENT record at all */
proc sql;
  select distinct weight
  from bdfs_am_final_summary_202412
  where sample_tp='CURRENT'
  order by weight;
quit;
