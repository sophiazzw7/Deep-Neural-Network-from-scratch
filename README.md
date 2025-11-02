/* Assume these exist from dev code */
%let min_posting_date = '01JAN2005'd;
%let max_posting_date = '31DEC2024'd;

%let BEFORE_DS = ops.operational_risk_loss_forecast;  /* raw Archer */
%let AFTER_DS  = ops.cleaned_severity_data;           /* event-level */

/* Quiet noisy procedures while we build result tables */
ods noresults; ods exclude all;

/* 1) ATTRIBUTES → small table */
proc contents data=&BEFORE_DS noprint out=qc_before_attr(keep=varnum name type length format);
run;
proc contents data=&AFTER_DS  noprint out=qc_after_attr (keep=varnum name type length format);
run;

/* 2) BASIC STATS (only N, NMISS, MEAN, STD, MIN, MAX) */
proc means data=&BEFORE_DS noprint;
  var 'GL Amount'n;
  output out=qc_before_stats n=n nmiss=nmiss mean=mean std=std min=min max=max;
run;

proc means data=&AFTER_DS noprint;
  var gross_loss net_loss;
  output out=qc_after_stats n=n_gross nmiss=nmiss_gross mean=mean_gross std=std_gross
                           min=min_gross max=max_gross
                           n(net_loss)=n_net nmiss(net_loss)=nmiss_net
                           mean(net_loss)=mean_net std(net_loss)=std_net
                           min(net_loss)=min_net max(net_loss)=max_net;
run;

/* Date ranges */
proc sql noprint;
  create table qc_before_dates as
  select min('Posting Date'n) format=date9. as min_date,
         max('Posting Date'n) format=date9. as max_date
  from &BEFORE_DS;

  create table qc_after_dates as
  select min(Posting_Date) format=date9. as min_date,
         max(Posting_Date) format=date9. as max_date
  from &AFTER_DS;
quit;

/* 3) MISSINGNESS (compact) */
proc means data=&BEFORE_DS n nmiss noprint;  /* numeric only */
  var _numeric_;
  output out=qc_before_miss_num n=_n nmiss=_nmiss;
run;

proc sql noprint;
  create table qc_before_miss_char as
  select name, sum(missing(vvaluex(name))) as nmiss
  from dictionary.columns
  where libname=upcase(scan("&BEFORE_DS",1,'.'))
    and memname=upcase(scan("&BEFORE_DS",2,'.'))
    and type='char'
  group by name;
quit;

/* AFTER missingness */
proc means data=&AFTER_DS n nmiss noprint;
  var _numeric_;
  output out=qc_after_miss_num n=_n nmiss=_nmiss;
run;

proc sql noprint;
  create table qc_after_miss_char as
  select name, sum(missing(vvaluex(name))) as nmiss
  from dictionary.columns
  where libname=upcase(scan("&AFTER_DS",1,'.'))
    and memname=upcase(scan("&AFTER_DS",2,'.'))
    and type='char'
  group by name;
quit;

/* 4) DUPLICATES by Internal Events */
proc sort data=&BEFORE_DS out=_b_nodup dupout=_b_dups nodupkey;
  by 'Internal Events'n;
run;
proc sort data=&AFTER_DS  out=_a_nodup dupout=_a_dups nodupkey;
  by 'Internal Events'n;
run;

proc sql noprint;
  create table qc_dups as
  select 'BEFORE' as stage length=6,
         (select count(*) from &BEFORE_DS)  as total_rows,
         (select count(*) from _b_nodup)    as unique_ids,
         (select count(*) from _b_dups)     as dup_rows
  union all
  select 'AFTER',
         (select count(*) from &AFTER_DS),
         (select count(*) from _a_nodup),
         (select count(*) from _a_dups);
quit;

/* 5) RANGE FLAGS (compact counts only) */
data _rng_before;
  set &BEFORE_DS(keep='Posting Date'n 'GL Amount'n);
  length flag $40;
  if 'Posting Date'n < &min_posting_date or 'Posting Date'n > &max_posting_date then flag='DATE_OUT_OF_RANGE';
  else if 'GL Amount'n < 0 then flag='NEG_GL_AMOUNT';
  if not missing(flag);
run;

proc sql noprint;
  create table qc_before_range as
  select flag, count(*) as cnt from _rng_before group by flag;
quit;

data _rng_after;
  set &AFTER_DS(keep=Posting_Date gross_loss net_loss);
  length flag $40;
  if Posting_Date < &min_posting_date or Posting_Date > &max_posting_date then flag='DATE_OUT_OF_RANGE';
  else if gross_loss < 0 then flag='NEG_GROSS_LOSS';
  else if net_loss   < 0 then flag='NEG_NET_LOSS';
  if not missing(flag);
run;

proc sql noprint;
  create table qc_after_range as
  select flag, count(*) as cnt from _rng_after group by flag;
quit;

/* 6) KEY CATEGORICALS → top 20 only (avoid huge output) */
proc freq data=&BEFORE_DS noprint;
  tables 'Basel Event Type Level 1'n 'GL Status'n 'Type of Impact'n 'Event Record Type'n / outpct out=qc_before_cats;
run;

proc sort data=qc_before_cats; by descending count; run;
data qc_before_cats; set qc_before_cats(obs=20); run;

proc freq data=&AFTER_DS noprint;
  tables 'Basel Event Type Level 1'n / outpct out=qc_after_cats;
run;
proc sort data=qc_after_cats; by descending count; run;

/* turn results back on */
ods results; ods exclude none;

/* ----------------- PRINT SMALL TABLES ONLY ------------------------ */
title "QC ROLLUP — Duplicates and Range Flags";
proc print data=qc_dups noobs; run;
proc print data=qc_before_range noobs; run;
proc print data=qc_after_range  noobs; run;

title "Attributes (first 30 variables only) — BEFORE / AFTER";
proc print data=qc_before_attr(obs=30) noobs; run;
proc print data=qc_after_attr (obs=30) noobs; run;

title "Descriptives — BEFORE ('GL Amount') and AFTER (gross_loss / net_loss)";
proc print data=qc_before_stats noobs; run;
proc print data=qc_after_stats  noobs; run;
proc print data=qc_before_dates noobs; run;
proc print data=qc_after_dates  noobs; run;

title "Missingness (numeric counts and top character fields)";
proc print data=qc_before_miss_num noobs; run;
proc print data=qc_after_miss_num  noobs; run;
proc print data=qc_before_miss_char(obs=15) noobs; run;
proc print data=qc_after_miss_char (obs=15) noobs; run;

title "Top 20 Category Levels — BEFORE / AFTER";
proc print data=qc_before_cats noobs; run;
proc print data=qc_after_cats  noobs; run;
title;
