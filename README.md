/*============================================================*
 *  Operational Risk Loss Forecast – DQ & Reasonableness Check
 *  (Aligned to developer pipeline & kept variables)
 *============================================================*/

/*--- Setup (edit the libref/path or dates if needed) ---*/
options mprint mlogic symbolgen;
libname ops "/sasdata/mrmg1/users/g07267/MOD13638/Input";

%let min_posting_date = '01JAN2005'd;
%let max_posting_date = '31DEC2024'd;

/*============================================================*
 * A) Replicate developer pre-clean steps (vars kept)
 *============================================================*/

/* Round GL Amount (developer creates GL_Amount) */
data cleaned_data;
  set ops.operational_risk_loss_forecast;
  GL_Amount = round('GL Amount'n, 0.01);
run;

/* Drop the same columns developer removed (thus “kept” = rest) */
data cleaned_data;
  set cleaned_data;
  drop
    'GL Account'n
    'Assigned Group'n
    'Profit Center'n
    'Origin'n
    'Source System'n
    'Reference'n
    'Event Creation Date'n
    'Basel Business Line 1'n
    'Basel Business Line 2'n
    'Event Start Date'n
    'Department'n
    'Business Unit'n
    'Litigation/Settlement'n
  ;
run;

/*============================================================*
 * B) Structure / Units / Formats inventory (post-drop)
 *============================================================*/
proc contents data=cleaned_data varnum
              out=work._dict_(keep=name type length format informat label) noprint;
run;

title "Variable Units & Formats (After Developer Drop)";
proc print data=work._dict_ noobs label;
  label name="Variable"
        type="Type (1=Num,2=Char)"
        length="Length"
        format="Format"
        informat="Informat"
        label="Label";
run;
title;

/*============================================================*
 * C) Basic Descriptive Statistics & Missing
 *============================================================*/

/* Numeric summaries */
title "Descriptive Statistics (Numeric Variables)";
proc means data=cleaned_data n nmiss mean std min p1 p5 p25 p50 p75 p95 p99 max maxdec=4;
  var _numeric_;
run;
title;

/* Character missing distribution (includes blanks if any) */
title "Missing Summary (Character Variables; shows missing row if any)";
proc freq data=cleaned_data;
  tables _character_ / missing nopercent nocum;
run;
title;

/*============================================================*
 * D) Duplicate checks on the event key (developer uses 'Internal Events'n)
 *============================================================*/
proc sort data=cleaned_data out=work._sorted_;
  by 'Internal Events'n;
run;

/* Any duplicate 'Internal Events' (same event id) */
proc sort data=work._sorted_ out=work._nodup_ nodupkey dupout=work._dups_;
  by 'Internal Events'n;
run;

title "Duplicate Check on 'Internal Events'n";
proc sql;
  select count(*) as total_rows format=comma18.
       , (select count(*) from work._dups_) as duplicate_rows format=comma18.
  from work._sorted_;
quit;

%if %sysfunc(exist(work._dups_)) %then %do;
  title "Sample Duplicate Rows (up to 50)";
  proc print data=work._dups_(obs=50) noobs;
  run;
%end;
title;

/*============================================================*
 * E) Reasonableness & Policy-style checks
 *    (mirrors developer exclusion intent but only FLAGS,
 *     it does NOT delete; you can review then decide)
 *============================================================*/
data work._checks_;
  set cleaned_data;

  length reason $800;
  retain reason;
  reason="";

  /* Date window consistency */
  if not missing('Posting Date'n) then do;
    if 'Posting Date'n > &max_posting_date then reason=catx('; ', reason, 'Posting Date > max_posting_date');
    if 'Posting Date'n < &min_posting_date then reason=catx('; ', reason, 'Posting Date < min_posting_date');
    /* Future-dated sanity */
    if 'Posting Date'n > today() then reason=catx('; ', reason, 'Posting Date in future');
  end;

  /* Developer-intent exclusions surfaced as flags */
  if strip('Event Record Type'n) in
     ('Grouped Event (all <10K)','Near Miss Event','Gain Event','Event with no GL Impact')
  then reason=catx('; ', reason, 'Event Record Type excluded by dev');

  if strip('Event Status'n) in ('Draft','Archive','Submit for Review')
  then reason=catx('; ', reason, 'Event Status excluded by dev');

  if strip('Type of Impact'n) in
     ('Credit Boundary','Batch entry with events <10K','Accrual or Reserve',
      'Reclassification or Reversal','Timing Loss Related','Non-ORE')
  then reason=catx('; ', reason, 'Type of Impact excluded by dev');

  /* GL status pending review (guardrail) */
  if lowcase('GL Status'n) = 'pending review'
  then reason=catx('; ', reason, 'GL Status pending review');

  /* Negatives vs impact type (developer deletes when GL_Amount<0 & Type=Hard Dollar) */
  if not missing(GL_Amount) and GL_Amount < 0 and strip('Type of Impact'n) = 'Hard Dollar'
  then reason=catx('; ', reason, 'GL_Amount < 0 with Hard Dollar');

  /* Reasonable magnitude (tunable) */
  if not missing(GL_Amount) and abs(GL_Amount) > 1e9
  then reason=catx('; ', reason, 'GL_Amount unreasonably large (>1e9)');

  /* Required fields for downstream aggregation */
  if missing('Posting Date'n)               then reason=catx('; ', reason, 'Missing Posting Date');
  if missing('Type of Impact'n)             then reason=catx('; ', reason, 'Missing Type of Impact');
  if missing('GL Status'n)                  then reason=catx('; ', reason, 'Missing GL Status');
  if missing('Internal Events'n)            then reason=catx('; ', reason, 'Missing Internal Events');
  if missing('Basel Event Type Level 1'n)   then reason=catx('; ', reason, 'Missing Basel Event Type Level 1');
  if missing('Event Record Type'n)          then reason=catx('; ', reason, 'Missing Event Record Type');
  if missing(GL_Amount)                     then reason=catx('; ', reason, 'Missing GL_Amount');

  /* Final flag */
  flag_unreasonable = (strip(reason) ne "");
run;

/* Summary of flags */
title "Reasonableness Check – Summary";
proc sql;
  select count(*) as total_records format=comma18.,
         sum(flag_unreasonable) as flagged_records format=comma18.,
         calculated flagged_records / calculated total_records format=percent8.2 as pct_flagged
  from work._checks_;
quit;

/* Examples & Top drivers */
title "Flagged Examples (first 100)";
proc print data=work._checks_(where=(flag_unreasonable=1) obs=100) noobs;
  var 'Internal Events'n 'Posting Date'n 'Event Status'n 'Event Record Type'n
      'Type of Impact'n 'GL Status'n GL_Amount reason;
  format 'Posting Date'n date9.;
run;

title "Top Flag Reasons";
proc freq data=work._checks_(where=(flag_unreasonable=1));
  tables reason / nocum nopercent;
run;
title;

/*============================================================*
 * F) Gross-loss < 10K screening diagnostics (non-destructive)
 *    (developer removes events whose total gross loss < 10K)
 *============================================================*/
proc sql;
  create table work._gross_loss as
  select 'Internal Events'n
       , sum(GL_Amount) as Gross_Loss
  from cleaned_data
  where GL_Amount >= 0
  group by 'Internal Events'n;
quit;

title "Events With Gross_Loss < 10,000 (for potential exclusion)";
proc sql;
  select count(*) as events_below_10k
  from work._gross_loss
  where Gross_Loss < 10000;
quit;

proc print data=work._gross_loss(where=(Gross_Loss < 10000)) noobs;
  var 'Internal Events'n Gross_Loss;
run;
title;

/*============================================================*
 * G) Compact export-ready tables (optional)
 *============================================================*/

/* Data dictionary vs (optional) expected formats (edit if you maintain expectations) */
data work._expected_formats;
  length name $32 expected_format $32;
  infile datalines dlm='|' truncover;
  input name :$32. expected_format :$32.;
datalines;
GL_Amount|COMMA14.2
Posting Date|DATE9.
Basel Event Type Level 1|$64.
Internal Events|$64.
Event Record Type|$64.
Event Status|$64.
Type of Impact|$64.
GL Status|$64.
;
run;

proc sql;
  create table work._format_compare as
  select a.name
       , a.format as actual_format
       , b.expected_format
       , (strip(coalescec(upcase(a.format),'')) = strip(upcase(b.expected_format))) as format_matches
  from work._dict_ a
  left join work._expected_formats b
    on upcase(a.name) = upcase(b.name)
  order by name;
quit;

title "Format Comparison (Actual vs Expected)";
proc print data=work._format_compare noobs;
  var name actual_format expected_format format_matches;
run;
title;

/*============================================================*
 * H) Quick health KPI dashboard (counts)
 *============================================================*/
title "Quick Health KPIs";
proc sql;
  select
    (select count(*) from cleaned_data)                                    as n_rows                  format=comma18.,
    (select count(*) from work._dups_)                                     as n_dup_event_ids         format=comma18.,
    (select sum(flag_unreasonable) from work._checks_)                     as n_flagged               format=comma18.,
    (select count(*) from work._gross_loss where Gross_Loss < 10000)       as n_events_below_10k      format=comma18.
  ;
quit;
title;

/* End of script */
