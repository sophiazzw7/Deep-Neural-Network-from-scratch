/* Rank reasons by material count within each quarter */
proc sql;
  create table abl_reason_ranked as
  select quarter,
         reason,
         obs,
         ovr_cnt,
         mat_cnt,
         mat_rate,
         monotonic() as rank_within_qtr
  from (
        select quarter,
               reason,
               obs,
               ovr_cnt,
               mat_cnt,
               mat_rate
        from abl_by_reason
        order by quarter, mat_cnt desc
       )
  ;
quit;

/* Keep only top 10 per quarter */
data abl_reason_top10;
  set abl_reason_ranked;
  by quarter;
  if rank_within_qtr <= 10;
run;

title "Material Override Reasons (Top 10 by Material Count within Quarter)";
proc print data=abl_reason_top10 noobs label;
  var quarter reason obs ovr_cnt mat_cnt mat_rate;
run;
