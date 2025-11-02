/*-----------------------------------------------------------
  (Optional) Library assignment – uncomment & edit if needed
-----------------------------------------------------------*/
/* libname ops "/sasdata/mrmg1/users/g07267/MOD13630/Input"; */

/* Developer window per screenshots */
%let min_posting_date = '01JAN2005'd;
%let max_posting_date = '31DEC2024'd;

/*==========================================
  1) Baseline profile of the RAW Archer feed
==========================================*/
title "RAW: Events and GL Line Counts";

/* Distinct events and total rows */
proc sql;
  create table raw_counts as
  select count(*)                                    as rows_before,
         count(distinct 'Internal Events'n)          as events_before
  from ops.operational_risk_loss_forecast;
quit;

proc print data=raw_counts noobs; run;

/* How many GL lines per event? */
proc sql;
  create table line_counts as
  select 'Internal Events'n,
         count(*) as line_count
  from ops.operational_risk_loss_forecast
  group by 'Internal Events'n
  order by line_count desc;
quit;

proc means data=line_counts n min p25 median p75 max;
  var line_count;
  title "RAW: Distribution of GL lines per Internal Event";
run;

/* Sample a few multi-line events */
proc sql outobs=50;
  create table sample_multiline as
  select a.*
  from ops.operational_risk_loss_forecast as a
  where a.'Internal Events'n in
        (select 'Internal Events'n from line_counts where line_count>1)
  order by a.'Internal Events'n, a.'Posting Date'n;
quit;

title "RAW: Sample GL lines for multi-line Internal Events";
proc print data=sample_multiline(obs=20) width=min;
run;

/* Negative GL Amounts by Type of Impact (raw) */
proc sql;
  create table raw_negatives as
  select strip('Type of Impact'n) as type_of_impact length=60,
         count(*)                  as n_rows,
         min('GL Amount'n)         as min_amt format=dollar32.,
         max('GL Amount'n)         as max_amt format=dollar32.
  from ops.operational_risk_loss_forecast
  where 'GL Amount'n < 0
  group by calculated type_of_impact
  order by n_rows desc;
quit;

title "RAW: Negative GL Amounts by Type of Impact";
proc print data=raw_negatives noobs width=min; run;

/* Out-of-range Posting Date (raw) */
proc sql;
  create table raw_oob_date as
  select count(*) as n_oob
  from ops.operational_risk_loss_forecast
  where 'Posting Date'n > &max_posting_date
     or 'Posting Date'n < &min_posting_date;
quit;

title "RAW: Out-of-range Posting Date counts (relative to developer window)";
proc print data=raw_oob_date noobs; run;

/*=============================================================
  2) Reproduce developer cleaning to event-level (mirror logic)
     Source: developer SAS in your screenshots (DataCleaning.sas)
==============================================================*/

/* Step 2.1: Start with raw and round GL Amount to cents (as shown) */
data _cleaned_data;
  set ops.operational_risk_loss_forecast;
  'GL Amount'n = round('GL Amount'n, 0.01);
run;

/* Step 2.2: Drop the same descriptive fields (developer did a DROP) */
data _cleaned_data;
  set _cleaned_data;
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

/* Step 2.3: Exclusions with Impact (developer logic mirrored) */
data _data_with_exclusions;
  set _cleaned_data;

  /* Posting date window */
  if 'Posting Date'n > &max_posting_date then delete;

  /* Event Record Type not allowed */
  if strip('Event Record Type'n) in
      ('Grouped Event (all <10K)',
       'Near Miss Event',
       'Gain Event',
       'Event with no GL impact') then delete;

  /* Event Status must be Published/Completed */
  if strip('Event Status'n) in ('Draft','Archive','Submit for Review') then delete;

  /* Type of Impact categories excluded */
  if strip('Type of Impact'n) in
      ('Credit Boundary',
       'Batch entry with events <10K',
       'Accrual or Reserve',
       'Reclassification or Reversal',
       'Timing Loss Related',
       'Non-ORE') then delete;

  /* Negative "Hard Dollar" entries excluded */
  if ('GL Amount'n < 0) and (strip('Type of Impact'n) in ('Hard Dollar')) then delete;
run;

/* Step 2.4: Guardrail exclusions (developer logic mirrored) */
data _data_with_exclusions;
  set _data_with_exclusions;
  if lowcase('GL Status'n) = 'pending review' then delete;
  if missing('Posting Date'n) then delete;
  if missing('Type of Impact'n) then delete;
  if missing('GL Status'n) then delete;
  if missing('Internal Events'n) then delete;
  if missing('Basel Event Type Level 1'n) then delete;
  if missing('Event Record Type'n) then delete;
  if missing('GL Amount'n) then delete;
run;

/* Step 2.5: Compute event-level Gross Loss to enforce <10K exclusion */
proc sql;
  create table _gross_loss as
  select 'Internal Events'n,
         sum('GL Amount'n) as Gross_Loss
  from _data_with_exclusions
  where 'GL Amount'n >= 0
  group by 'Internal Events'n;
quit;

/* Remove events with Gross_Loss < 10,000 */
proc sql;
  create table _final_df as
  select *
  from _data_with_exclusions
  where 'Internal Events'n not in
        (select 'Internal Events'n from _gross_loss where Gross_Loss < 10000);
quit;

/* (Optional) Join the gross_loss back (developer did this) */
proc sql;
  create table _final_df as
  select a.*, b.Gross_Loss
  from _final_df as a
  left join _gross_loss as b
    on a.'Internal Events'n = b.'Internal Events'n;
quit;

/* Step 2.6: Roll-up to EVENT level (severity dataset) */
proc sql;
  create table work.cleaned_severity_data as
  select  'Internal Events'n,
          'Basel Event Type Level 1'n,
          min('Posting Date'n)                            as Posting_Date format=date9.,
          round(sum('GL Amount'n), 0.01)                  as net_loss,
          round(min(Gross_Loss), 0.01)                    as gross_loss
  from _final_df
  group by 'Internal Events'n, 'Basel Event Type Level 1'n;
quit;

/* Exclude net_loss < 0 at event level (developer did this) */
proc sql;
  create table work.cleaned_severity_data as
  select *
  from work.cleaned_severity_data
  where net_loss >= 0;
quit;

/*======================================================
  3) After-cleaning verification vs. RAW Archer extract
======================================================*/
title "AFTER CLEANING: One row per event?";

/* Distinct events and rows after cleanup */
proc sql;
  create table after_counts as
  select count(*)                           as rows_after,
         count(distinct 'Internal Events'n) as events_after
  from work.cleaned_severity_data;
quit;

proc print data=after_counts noobs; run;

/* Should be rows_after = events_after if each event collapses to 1 row */
proc sql;
  select (select events_before from raw_counts)  as events_before,
         (select rows_before   from raw_counts)  as rows_before,
         (select events_after  from after_counts) as events_after,
         (select rows_after    from after_counts) as rows_after
  ;
quit;

/* Confirm no residual duplicated Internal Events in cleaned severity */
proc sort data=work.cleaned_severity_data out=_sev_nodup dupout=_sev_dups nodupkey;
  by 'Internal Events'n;
run;

title "Check residual duplicates at event-level (should be 0)";
proc sql; select count(*) as dups_after from _sev_dups; quit;

/* Negative amounts by Type of Impact AFTER cleaning at line-level (for sanity) */
proc sql;
  create table cleaned_line_negatives as
  select strip('Type of Impact'n) as type_of_impact length=60,
         count(*)                  as n_rows,
         min('GL Amount'n)         as min_amt format=dollar32.,
         max('GL Amount'n)         as max_amt format=dollar32.
  from _final_df
  where 'GL Amount'n < 0
  group by calculated type_of_impact
  order by n_rows desc;
quit;

title "Cleaned LINE-level: Negative GL Amounts by Type of Impact (post-guardrails)";
proc print data=cleaned_line_negatives noobs width=min; run;

/*======================================================
  4) Decision helpers / flags you can cite in the report
======================================================*/

/* Events where multi-line structure existed in raw */
proc sql;
  create table events_multiline as
  select count(*) as n_events_multiline
  from line_counts
  where line_count>1;
quit;

title "Helper counts";
proc sql;
  select *
  from events_multiline;
quit;

/* Out-of-window rows removed count (raw) */
proc sql;
  create table raw_oob_details as
  select *
  from ops.operational_risk_loss_forecast
  where 'Posting Date'n > &max_posting_date
     or 'Posting Date'n < &min_posting_date;
quit;

/* Show how many such rows existed (context for 5k+ in your flag table) */
proc sql; select count(*) as raw_oob_rows from raw_oob_details; quit;

/* Clean up titles */
title;
