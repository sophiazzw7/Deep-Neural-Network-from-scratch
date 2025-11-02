/* ------------------------------------------------------------------ */
/* Assumes the developer's libname and date macros are already set:   */
/* libname ops "/sasdata/mrmg1/users/g07267/MOD13638/Input";          */
/* %let min_posting_date = '01JAN2005'd;                               */
/* %let max_posting_date = '31DEC2024'd;                               */
/* ------------------------------------------------------------------ */

%let BEFORE_DS = ops.operational_risk_loss_forecast;  /* raw Archer extract */
%let AFTER_DS  = ops.cleaned_severity_data;           /* developer’s event-level output */

/* ------------------------ 0) ATTRIBUTES ---------------------------- */
title "BEFORE: Variable Attributes (PROC CONTENTS)";
proc contents data=&BEFORE_DS varnum; run;

title "AFTER: Variable Attributes (PROC CONTENTS)";
proc contents data=&AFTER_DS varnum; run;
title;

/* Optional reporting formats (display only; does not alter stored values) */
proc datasets nolist;
  modify &BEFORE_DS;
    format 'Posting Date'n date9. 'GL Amount'n comma24.2;
  modify &AFTER_DS;
    format Posting_Date date9. gross_loss comma24.2 net_loss comma24.2;
quit;

/* ---------------- 1) BASIC DESCRIPTIVES (NUMERIC) ------------------ */
/* BEFORE: GL Amount */
title "BEFORE: Descriptives for 'GL Amount'n";
proc means data=&BEFORE_DS n nmiss mean std min p1 p5 p50 p95 max maxdec=2;
  var 'GL Amount'n;
run;
/* BEFORE: date min/max for 'Posting Date'n */
title "BEFORE: Date Range for 'Posting Date'n";
proc sql;
  select min('Posting Date'n) format=date9. as min_date,
         max('Posting Date'n) format=date9. as max_date
  from &BEFORE_DS;
quit;

/* AFTER: gross_loss / net_loss */
title "AFTER: Descriptives for gross_loss and net_loss";
proc means data=&AFTER_DS n nmiss mean std min p1 p5 p50 p95 max maxdec=2;
  var gross_loss net_loss;
run;
/* AFTER: date min/max for Posting_Date */
title "AFTER: Date Range for Posting_Date";
proc sql;
  select min(Posting_Date) format=date9. as min_date,
         max(Posting_Date) format=date9. as max_date
  from &AFTER_DS;
quit;
title;

/* --------------------- 2) MISSINGNESS ------------------------------ */
proc format;
  value missnum . = 'MISSING' other='NOT MISSING';
  value $misschr ' ' = 'MISSING' other='NOT MISSING';
run;

/* BEFORE: numeric missingness */
title "BEFORE: Missingness - Numeric Variables";
proc means data=&BEFORE_DS n nmiss;
  var _numeric_;
run;
/* BEFORE: character missingness (frequency with MISSING option) */
title "BEFORE: Missingness - Character Variables";
proc freq data=&BEFORE_DS;
  tables _character_ / missing;
run;

/* AFTER: numeric missingness */
title "AFTER: Missingness - Numeric Variables";
proc means data=&AFTER_DS n nmiss;
  var _numeric_;
run;
/* AFTER: character missingness */
title "AFTER: Missingness - Character Variables";
proc freq data=&AFTER_DS;
  tables _character_ / missing;
run;
title;

/* ------------------- 3) DUPLICATION CHECKS ------------------------ */
/* Duplicate check key = 'Internal Events'n (matches developer code) */
title "BEFORE: Duplicate Check by 'Internal Events'n";
proc sort data=&BEFORE_DS out=_before_nodup dupout=_before_dups nodupkey;
  by 'Internal Events'n;
run;
proc sql;
  select (select count(*) from &BEFORE_DS)      as total_rows,
         (select count(*) from _before_nodup)   as unique_ids,
         (select count(*) from _before_dups)    as duplicate_rows;
quit;

/* AFTER: there should be one row per event id */
title "AFTER: Duplicate Check by 'Internal Events'n";
proc sort data=&AFTER_DS out=_after_nodup dupout=_after_dups nodupkey;
  by 'Internal Events'n;
run;
proc sql;
  select (select count(*) from &AFTER_DS)      as total_rows,
         (select count(*) from _after_nodup)   as unique_ids,
         (select count(*) from _after_dups)    as duplicate_rows;
quit;
title;

/* --------------- 4) RANGE REASONABLENESS CHECKS ------------------- */
/* BEFORE: flag out-of-range dates and negative GL Amounts */
data _rng_before;
  set &BEFORE_DS;
  length range_flag $200;
  range_flag = '';
  if 'Posting Date'n < &min_posting_date or 'Posting Date'n > &max_posting_date
     then range_flag = catx('|', range_flag, 'DATE_OUT_OF_RANGE');
  if 'GL Amount'n < 0 then
     range_flag = catx('|', range_flag, 'NEG_GL_AMOUNT');
  if not missing(range_flag);
run;

title "BEFORE: Range Flags (Posting Date, GL Amount)";
proc freq data=_rng_before;
  tables range_flag / nocum;
run;

/* AFTER: flag out-of-range Posting_Date and negative net_loss/gross_loss */
data _rng_after;
  set &AFTER_DS;
  length range_flag $200;
  range_flag = '';
  if Posting_Date < &min_posting_date or Posting_Date > &max_posting_date
     then range_flag = catx('|', range_flag, 'DATE_OUT_OF_RANGE');
  if net_loss < 0 then range_flag = catx('|', range_flag, 'NEG_NET_LOSS');
  if gross_loss < 0 then range_flag = catx('|', range_flag, 'NEG_GROSS_LOSS');
  if not missing(range_flag);
run;

title "AFTER: Range Flags (Posting_Date, net_loss, gross_loss)";
proc freq data=_rng_after;
  tables range_flag / nocum;
run;
title;

/* --------------- 5) SIMPLE VALUE LISTING (CATEGORICALS) ----------- */
/* BEFORE: list values and counts for main categoricals used by dev */
title "BEFORE: Value Listing - Key Categorical Variables";
proc freq data=&BEFORE_DS;
  tables 'Basel Event Type Level 1'n
         'GL Status'n
         'Type of Impact'n
         'Event Record Type'n / missing;
run;

/* AFTER: corresponding variables present in event-level output */
title "AFTER: Value Listing - Key Categorical Variables";
proc freq data=&AFTER_DS;
  tables 'Basel Event Type Level 1'n / missing;
run;
title;

/* ------------------ 6) ONE-PAGE QC ROLLUP ------------------------- */
title "QC ROLLUP: Before vs After";
proc sql;
  select
    (select count(*) from &BEFORE_DS)      as before_rows,
    (select count(*) from _before_dups)    as before_dup_rows,
    (select count(*) from _rng_before)     as before_range_flags,
    (select count(*) from &AFTER_DS)       as after_rows,
    (select count(*) from _after_dups)     as after_dup_rows,
    (select count(*) from _rng_after)      as after_range_flags;
quit;
title;
