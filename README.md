proc sql;
select count(*) as neg_scores
from bdfs_am_final_summary_202412
where sample_tp='CURRENT' and score_value < 100;
quit;

