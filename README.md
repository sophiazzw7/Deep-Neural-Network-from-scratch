/* ---------- Params ---------- */
%let rpt_prd_date_id = 20241231;
/* 12 months before report EOM: 31DEC2023 (numeric SAS date) */
%let eop12n = %sysfunc(intnx(month,%sysfunc(inputn(&rpt_prd_date_id,yymmdd8.)),-12,end));

/* ---------- Build the flat slice (CURRENT, start_date <= eop12) ---------- */
/* start_date = EOM(score_date); fallback to eff_from_dt if score_date missing */
data _flat_expected;
  set model.f_app_score_flat;
  length start_date 8; format start_date date9.;
  start_date = coalesce(intnx('month', score_date, 0, 'end'), eff_from_dt);
  if sample_tp='CURRENT' and start_date <= &eop12n;
  keep company_id start_date score_date eff_from_dt score_value weight sample_tp unique_id;
run;

/* Quick sanity */
proc sql;
  select count(*) as n_flat_expected,
         min(start_date) format=date9. as min_start,
         max(start_date) format=date9. as max_start
  from _flat_expected;
quit;

/* ---------- Duplicates in the flat slice ---------- */
/* Prefer unique_id if it’s truly 1–1; otherwise fall back to (company_id,start_date) */
%macro check_dupes_flat;
  %if %sysfunc(varnum(%sysfunc(open(_flat_expected)),unique_id)) > 0 %then %do;
    proc sort data=_flat_expected out=_fe_uid_dedup dupout=_fe_uid_dups nodupkey;
      by unique_id;
    run;
    title "Duplicates in _flat_expected on unique_id";
    proc sql; select count(*) as dup_rows from _fe_uid_dups; quit;
    proc print data=_fe_uid_dups(obs=20); var unique_id company_id start_date score_value weight; run;
  %end;
  %else %do;
    proc sort data=_flat_expected out=_fe_dedup dupout=_fe_dups nodupkey;
      by company_id start_date;
    run;
    title "Duplicates in _flat_expected on (company_id, start_date)";
    proc sql; select count(*) as dup_rows from _fe_dups; quit;
    proc print data=_fe_dups(obs=20); var company_id start_date score_value weight; run;
  %end;
%mend;
%check_dupes_flat;
