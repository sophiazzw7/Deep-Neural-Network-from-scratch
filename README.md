/* ============================== */
/* 0) Setup                       */
/* ============================== */
ods graphics on;
options mprint nomlogic;
title; footnote;

%let rpt_prd_date_id = 20241231;
%let eop12 = %sysfunc(intnx(month,%sysfunc(inputn(&rpt_prd_date_id,yymmdd8.)),-12,end),date9.);
%put NOTE: eop12 cutoff for CURRENT = &eop12;  /* expect 31DEC2023 */

/* Libs assumed:
   - cubes.bdfs_am_final_summary_202412
   - model.f_app_score_flat
   - work.intime_sample_30jun2015  (change lib if needed) */

/* ============================== */
/* 1) High-level structure & counts */
/* ============================== */
title "Structure: cubes.bdfs_am_final_summary_202412";
proc contents data=cubes.bdfs_am_final_summary_202412 order=varnum; run;

title "Basic Counts and Date Ranges (Final Cube)";
proc sql;
  select count(*) as total_rows
       , sum(sample_tp='CURRENT') as n_current
       , sum(sample_tp='INTIME')  as n_intime
       , min(start_date) format=date9. as earliest_start
       , max(start_date) format=date9. as latest_start
       , min(rpt_prd_date_id) as earliest_rpt
       , max(rpt_prd_date_id) as latest_rpt
  from cubes.bdfs_am_final_summary_202412;
quit;

title "Vintage Mix by Sample Type (Final Cube)";
proc sql;
  select sample_tp, vintage_yr, count(*) as n
  from cubes.bdfs_am_final_summary_202412
  group by sample_tp, vintage_yr
  order by sample_tp, vintage_yr;
quit;

/* ============================== */
/* 2) Create OGM slice & helpers  */
/* ============================== */
title "Building 202412 OGM Slices (Aligned to ETL)";
data _cube_current; set cubes.bdfs_am_final_summary_202412;
  where sample_tp='CURRENT' and start_date <= "&eop12"d;
run;

data _cube_intime; set cubes.bdfs_am_final_summary_202412;
  where sample_tp='INTIME';
run;

data _flat_expected; set model.f_app_score_flat;
  where rpt_prd_date_id=&rpt_prd_date_id and start_date <= "&eop12"d;
run;

title "Sanity: Counts After Slicing";
proc sql;
  select 'cube_current' as table, count(*) as n from _cube_current
  union all select 'cube_intime', count(*) from _cube_intime
  union all select 'flat_expected', count(*) from _flat_expected;
quit;

/* Overlap check between cube CURRENT and score-flat using ETL keys */
title "Overlap: CURRENT (cube) vs Score-Flat (expected) on (company_id,start_date)";
proc sql;
  select 'cube_n' as metric length=16, count(*) as value from _cube_current
  union all
  select 'flat_n', count(*) from _flat_expected
  union all
  select 'matched_keys', count(*)
  from (
        select company_id, start_date from _cube_current
        intersect
        select company_id, start_date from _flat_expected
       );
quit;

/* ============================== */
/* 3) Duplicate checks (ETL keys) */
/* ============================== */
title "Duplicate Check on ETL Keys (company_id, start_date, lob_indicator, sample_tp)";
proc sort data=cubes.bdfs_am_final_summary_202412 out=_dedup dupout=_dups nodupkey;
  by company_id start_date lob_indicator sample_tp;
run;
proc sql; select count(*) as dups_on_etl_keys from _dups; quit;

proc print data=_dups(obs=20) label;
  title2 "Sample of duplicate rows (if any)";
run;

/* ============================== */
/* 4) Distributions & reasonableness */
/* ============================== */
title "Score & Weight Summary (Final Cube)";
proc means data=cubes.bdfs_am_final_summary_202412 n nmiss min p1 p5 p10 q1 median mean q3 p90 p95 p99 max std;
  var score_value weight;
run;

title "Histogram: Model Score (score_value) - Final Cube";
proc sgplot data=cubes.bdfs_am_final_summary_202412;
  histogram score_value / nbins=40;
  density score_value;
  xaxis label="Model Score (score_value)";
  yaxis label="Frequency";
run;

title "Histogram: Record Weight (weight) - Final Cube";
proc sgplot data=cubes.bdfs_am_final_summary_202412;
  histogram weight;
  density weight;
  xaxis label="Record Weight";
  yaxis label="Frequency";
run;

/* Event rates by vintage_yr/sample_tp */
title "Event Rates by Sample Type and Vintage Year (Final Cube)";
proc sql;
  select sample_tp, vintage_yr,
         mean(BAD_AT_3MO)  format=percent8.2 as p_bad_3m,
         mean(BAD_AT_6MO)  format=percent8.2 as p_bad_6m,
         mean(BAD_AT_12MO) format=percent8.2 as p_bad_12m,
         mean(EVER_BAD)    format=percent8.2 as p_ever_bad
  from cubes.bdfs_am_final_summary_202412
  group by sample_tp, vintage_yr
  order by sample_tp, vintage_yr;
quit;

/* Correlation among bad flags should increase with horizon */
title "Correlation Among BAD Flags (Final Cube)";
proc corr data=cubes.bdfs_am_final_summary_202412 nosimple;
  var BAD_AT_0MO BAD_AT_3MO BAD_AT_6MO BAD_AT_9MO BAD_AT_12MO EVER_BAD;
run;

/* LOB coverage */
title "LOB Coverage by Sample Type (Final Cube)";
proc freq data=cubes.bdfs_am_final_summary_202412;
  tables lob_indicator*sample_tp / missing norow nocol nopercent;
run;

/* ============================== */
/* 5) Reasonableness & outlier flags */
/*     (focus: CURRENT; INTIME reported separately) */
/* ============================== */
data _issues_current _issues_intime;
  set cubes.bdfs_am_final_summary_202412;
  length issue $60;
  if sample_tp='CURRENT' then do;
    if score_value < 0 then issue='Negative score (CURRENT)';
    if weight <= 0 then issue='Non-positive weight (CURRENT)';
    if BAD_AT_12MO not in (0,1,.) then issue='Invalid BAD_AT_12MO (CURRENT)';
    if not missing(issue) then output _issues_current;
  end;
  else if sample_tp='INTIME' then do;
    if score_value < 100 then issue='Suspicious INTIME score (placeholder?)';
    if not missing(issue) then output _issues_intime;
  end;
run;

title "Potential Data Issues (CURRENT)";
proc freq data=_issues_current; tables issue; run;
proc print data=_issues_current(obs=20); run;

title "Potential Data Issues (INTIME)";
proc freq data=_issues_intime; tables issue; run;
proc print data=_issues_intime(obs=20); run;

/* ============================== */
/* 6) Time sanity (start vs rpt)  */
/* ============================== */
title "Time Range Check (Final Cube)";
proc sql;
  select min(start_date) format=date9. as earliest_start,
         max(start_date) format=date9. as latest_start,
         min(rpt_prd_date_id) as earliest_rpt,
         max(rpt_prd_date_id) as latest_rpt
  from cubes.bdfs_am_final_summary_202412;
quit;

/* ============================== */
/* 7) INTIME presence cross-check */
/* ============================== */
title "INTIME Presence & Matching (by company_id)";
proc sql;
  select count(*) as intime_in_cube
  from cubes.bdfs_am_final_summary_202412
  where sample_tp='INTIME';

  /* change WORK. if your INTIME table is in another lib */
  select count(*) as intime_matched
  from cubes.bdfs_am_final_summary_202412 a
  inner join work.intime_sample_30jun2015 b
    on a.company_id=b.company_id
  where a.sample_tp='INTIME';
quit;

/* ============================== */
/* 8) Final summary snapshot      */
/* ============================== */
title "Summary Snapshot: CURRENT slice (<= &eop12)";
proc means data=_cube_current n nmiss min p1 p5 q1 median mean q3 p95 p99 max;
  var score_value weight BAD_AT_3MO BAD_AT_6MO BAD_AT_12MO;
run;

title "Vintage Mix in CURRENT slice (should be mostly 2023)";
proc sql;
  select vintage_yr, count(*) as n
  from _cube_current
  group by vintage_yr
  order by vintage_yr;
quit;

ods graphics off;
title; footnote;
