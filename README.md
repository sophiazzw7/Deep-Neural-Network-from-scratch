/* =========================
   0) Parameters & helpers
   ========================= */
ods graphics on;
options mprint;
title; footnote;

%let rpt_prd_date_id = 20241231;
/* End-of-month 12 months before report date = ETL's eop12 */
%let eop12 = %sysfunc(intnx(month,%sysfunc(inputn(&rpt_prd_date_id,yymmdd8.)),-12,end),date9.);
%put NOTE: 2024Q4 CURRENT cutoff (eop12) = &eop12;  /* -> 31DEC2023 */

/* =========================
   1) Build ETL-consistent slices
   ========================= */
/* FINAL cube: keep exactly what ETL uses for 2024Q4 */
data _cube_current  _cube_intime;
  set cubes.bdfs_am_final_summary_202412;
  /* Cube already includes INTIME; keep the ETL slicer for CURRENT */
  if sample_tp='CURRENT' and start_date <= "&eop12"d then output _cube_current;
  else if sample_tp='INTIME' then output _cube_intime;
run;

/* f_app_score_flat: reconstruct START_DATE from SCORE_DATE (end-of-month),
   standardize COMPANY_ID to text for matching */
data _flat_expected;
  set model.f_app_score_flat;
  length company_id_txt $32;
  company_id_txt = strip(company_id);                 /* char in your file */
  length score_dt start_date 8; format score_dt start_date date9.;
  score_dt   = score_date;                            /* already DATE9. */
  start_date = intnx('month', score_dt, 0, 'end');    /* ETL does this */
  where sample_tp='CURRENT' and start_date <= "&eop12"d;
run;

/* For the cube, standardize the join key to the same text+date format */
data _cube_current_keys(keep=key) _flat_expected_keys(keep=key);
  length key $64;
  set _cube_current(in=a) _flat_expected(in=b);
  if a then key = cats(strip(put(company_id,best.)),'|',put(start_date,yymmddn8.));
  else if b then key = cats(strip(company_id_txt),'|',put(start_date,yymmddn8.));
  if a then output _cube_current_keys;
  if b then output _flat_expected_keys;
run;

/* =========================
   2) Lineage sanity: counts & overlap
   ========================= */
proc sql;
  title "Population: 2024Q4 OGM slices";
  select 'cube_current' as table length=20, count(*) as n from _cube_current
  union all select 'cube_intime', count(*) from _cube_intime
  union all select 'flat_expected', count(*) from _flat_expected;

  title "Overlap (company_id, start_date): cube CURRENT vs f_app EXPECTED";
  select 'cube_n' as metric length=16, count(*) as value from _cube_current_keys
  union all select 'flat_n', count(*) from _flat_expected_keys
  union all select 'matched_keys', count(*) 
  from (select key from _cube_current_keys
        intersect
        select key from _flat_expected_keys);
quit;

/* Mismatches for investigation */
proc sql;
  create table _in_cube_not_flat as
  select a.*
  from _cube_current a
  left join _flat_expected b
    on cats(strip(put(a.company_id,best.)),'|',put(a.start_date,yymmddn8.))
     = cats(strip(b.company_id_txt),'|',put(b.start_date,yymmddn8.))
  where b.company_id_txt is null;

  create table _in_flat_not_cube as
  select b.*
  from _flat_expected b
  left join _cube_current a
    on cats(strip(put(a.company_id,best.)),'|',put(a.start_date,yymmddn8.))
     = cats(strip(b.company_id_txt),'|',put(b.start_date,yymmddn8.))
  where a.company_id is null;
quit;

title "Sample: in cube CURRENT but not in flat (first 20)";
proc print data=_in_cube_not_flat(obs=20); run;

title "Sample: in flat EXPECTED but not in cube CURRENT (first 20)";
proc print data=_in_flat_not_cube(obs=20); run;

/* =========================
   3) Duplicate checks
   ========================= */
/* In cube – use ETL keys (company_id, start_date, lob_indicator, sample_tp) */
proc sort data=_cube_current out=_cube_current_dedup dupout=_cube_current_dups nodupkey;
  by company_id start_date lob_indicator sample_tp;
run;

title "Duplicates in cube CURRENT on ETL keys";
proc sql; select count(*) as dup_rows from _cube_current_dups; quit;
proc print data=_cube_current_dups(obs=20); run;

/* In f_app – closest analog (company_id, start_date) */
proc sort data=_flat_expected out=_flat_expected_dedup dupout=_flat_expected_dups nodupkey;
  by company_id_txt start_date;
run;

title "Duplicates in f_app EXPECTED on (company_id, start_date)";
proc sql; select count(*) as dup_rows from _flat_expected_dups; quit;
proc print data=_flat_expected_dups(obs=20); run;

/* =========================
   4) Logical checks (light but targeted)
   ========================= */
title "Score & Weight Summary – cube CURRENT (<= &eop12)";
proc means data=_cube_current n nmiss min p1 p5 q1 median mean q3 p95 p99 max std;
  var score_value weight;
run;

title "Event Rates by vintage_yr – cube CURRENT";
proc sql;
  select vintage_yr,
         mean(BAD_AT_3MO)  format=percent8.2 as p_bad_3m,
         mean(BAD_AT_6MO)  format=percent8.2 as p_bad_6m,
         mean(BAD_AT_12MO) format=percent8.2 as p_bad_12m,
         mean(EVER_BAD)    format=percent8.2 as p_ever
  from _cube_current
  group by vintage_yr
  order by vintage_yr;
quit;

title "Reasonableness flags – cube CURRENT";
data _issues_current;
  set _cube_current;
  length issue $60;
  if score_value < 0 then issue='Negative score';
  else if score_value > 1000 then issue='Score > 1000';     /* adjust if your scale is different */
  else if weight <= 0 then issue='Non-positive weight';
  else if BAD_AT_12MO not in (0,1,.) then issue='Invalid BAD_AT_12MO';
  if not missing(issue);
run;

proc freq data=_issues_current; tables issue; run;
proc print data=_issues_current(obs=20); run;

/* Quick time bounds */
title "Time bounds – cube CURRENT";
proc sql;
  select min(start_date) format=date9. as earliest_start,
         max(start_date) format=date9. as latest_start
  from _cube_current;
quit;

ods graphics off;
title; footnote;
