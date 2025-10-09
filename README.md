proc contents data=bdfs_am_final_summary_202412; run;
proc sql;
  select count(*) as nobs,
         count(distinct company_id) as n_company,
         count(distinct unique_id) as n_unique,
         min(start_date) as min_start format=date9.,
         max(start_date) as max_start format=date9.,
         min(rpt_prd_date_id) as min_rpt,
         max(rpt_prd_date_id) as max_rpt,
         count(distinct vintage_yr) as vintages
  from bdfs_am_final_summary_202412;
quit;

proc freq data=bdfs_am_final_summary_202412;
  tables sample_tp vintage_yr / nocum nopercent;
run;

proc means data=bdfs_am_final_summary_202412 n nmiss min p1 p5 p10 q1 median mean q3 p90 p95 p99 max std;
  var score_value weight;
run;
proc sgplot data=bdfs_am_final_summary_202412;
  histogram score_value / nbins=40;
  density score_value;
  title "Score Distribution";
run;
proc sql;
  select sample_tp, vintage_yr,
         mean(BAD_AT_3MO) as p3m format=percent8.2,
         mean(BAD_AT_6MO) as p6m format=percent8.2,
         mean(BAD_AT_12MO) as p12m format=percent8.2,
         mean(EVER_BAD) as p_ever format=percent8.2
  from bdfs_am_final_summary_202412
  group by sample_tp, vintage_yr;
quit;



proc means data=bdfs_am_final_summary_202412 min max mean nmiss;
  var weight;
run;

proc sort data=bdfs_am_final_summary_202412 nodupkey dupout=dq_dups;
  by sample_tp vintage_yr unique_id;
run;
proc sql; select count(*) as duplicate_rows from dq_dups; quit;
proc corr data=bdfs_am_final_summary_202412;
  var BAD_AT_0MO BAD_AT_3MO BAD_AT_6MO BAD_AT_9MO BAD_AT_12MO EVER_BAD;
run;
proc freq data=bdfs_am_final_summary_202412;
  tables lob_indicator*sample_tp / missing;
run;
proc sql;
  select min(start_date) as earliest_start format=date9.,
         max(start_date) as latest_start format=date9.,
         min(rpt_prd_date_id) as earliest_rpt,
         max(rpt_prd_date_id) as latest_rpt
  from bdfs_am_final_summary_202412;
quit;
data dq_outliers;
  set bdfs_am_final_summary_202412;
  length issue $50;
  if score_value < 0 then issue='Negative score';
  if score_value > 1000 then issue='Score>1000';
  if weight <= 0 then issue='Non-positive weight';
  if BAD_AT_12MO not in (0,1,.) then issue='Invalid BAD flag';
  if not missing(issue) then output;
run;

proc print data=dq_outliers (obs=20); run;
