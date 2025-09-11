/* Build reason-level summary with repeat info */
proc sql;
  create table abl_reason_by_qtr as
  select quarter,
         Override_Reason as reason,
         count(*)                        as obs,
         sum(material_1notch)            as mat_cnt,
         sum(repeat_material)            as mat_repeat,
         sum(same_reason_repeat)         as same_reason_repeat,
         calculated same_reason_repeat /
           max(calculated mat_repeat,1)  as same_reason_pct format=percent8.2,
         sum(material_1notch) /
           calculated obs                as mat_rate format=percent8.2
  from _reason_seq
  group by quarter, Override_Reason
  order by quarter, mat_cnt desc;
quit;

/* Rank within each quarter and keep top 10 */
proc sort data=abl_reason_by_qtr out=abl_reason_sorted;
    by quarter descending mat_cnt reason;
run;

data abl_reason_top10;
    set abl_reason_sorted;
    by quarter;
    retain rank_within_qtr;
    if first.quarter then rank_within_qtr=0;
    rank_within_qtr+1;
    if rank_within_qtr<=10;
run;

/* Print the enriched top 10 reason table */
title "Material Override Reasons (Top 10 by Material Count within Quarter, incl. Repeat Info)";
proc print data=abl_reason_top10 noobs label;
    var quarter reason obs mat_cnt mat_repeat same_reason_repeat same_reason_pct mat_rate;
run;
title;
