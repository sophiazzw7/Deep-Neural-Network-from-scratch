/*==========================================
 * QA: Event-level + Quarterly model inputs
 *   - ops.cleaned_severity_data
 *   - aggregated_frequency
 *==========================================*/

%let MAX_PRINT = 20;

/*------------------------------------------
 | A) Event-level: ops.cleaned_severity_data
 *------------------------------------------*/

/* 1) Shape & required fields present */
title "cleaned_severity_data — Variable Inventory";
proc contents data=ops.cleaned_severity_data varnum noprint
              out=work._dict_csd(keep=name type length format label);
run;
proc print data=work._dict_csd noobs; where name in
 ("Internal Events","Basel Event Type Level 1","Posting_Date","data_date","net_loss","gross_loss"); run; title;

/* 2) Missing counts (concise) */
title "cleaned_severity_data — Missing Summary";
proc means data=ops.cleaned_severity_data n nmiss;
  var net_loss gross_loss Posting_Date data_date;
run; title;

/* 3) Basic validity */
title "cleaned_severity_data — Validity Checks";
proc sql;
  /* Unique by Event × Basel L1 */
  create table work._dup_csd as
  select 'Internal Events'n, 'Basel Event Type Level 1'n, count(*) as n
  from ops.cleaned_severity_data
  group by 'Internal Events'n, 'Basel Event Type Level 1'n
  having calculated n>1;

  /* Summary flags */
  select
    count(*)                                as rows format=comma18.,
    (select count(*) from work._dup_csd)    as duplicate_event_l1_pairs format=comma18.,
    sum(net_loss<0)                         as n_neg_net_loss format=comma18.,
    sum(gross_loss<0)                       as n_neg_gross_loss format=comma18.,
    sum(net_loss>gross_loss)                as n_net_gt_gross format=comma18.,
    sum(Posting_Date>today())               as n_future_posting format=comma18.,
    sum(. = data_date)                      as n_missing_quarter format=comma18.
  from ops.cleaned_severity_data;
quit; title;

/* 4) Distributions & outlier scan (concise percentiles) */
title "cleaned_severity_data — Net/Gross Loss Percentiles";
proc means data=ops.cleaned_severity_data
           n mean std min p1 p5 p25 p50 p75 p95 p99 max maxdec=4;
  var net_loss gross_loss;
run; title;

/* (Optional) Top events by net_loss — capped to avoid large output */
title "cleaned_severity_data — Top Events by net_loss (sample)";
proc sort data=ops.cleaned_severity_data out=work._csd_top;
  by descending net_loss;
run;
proc print data=work._csd_top(obs=&MAX_PRINT) noobs;
  var 'Basel Event Type Level 1'n 'Internal Events'n Posting_Date net_loss gross_loss;
  format Posting_Date date9.;
run; title;

/*------------------------------------------------
 | B) Quarterly: aggregated_frequency (reconcile)
 *------------------------------------------------*/

/* 1) Columns present & missing/validity */
title "aggregated_frequency — Variable Inventory";
proc contents data=aggregated_frequency varnum noprint
              out=work._dict_af(keep=name type length format label);
run;
proc print data=work._dict_af noobs; where name in
 ("data_date","event_type","frequency","gross_loss","net_loss"); run; title;

title "aggregated_frequency — Missing / Basic Validity";
proc means data=aggregated_frequency n nmiss;
  var frequency gross_loss net_loss data_date;
run;
proc sql;
  select
    count(*)                         as rows format=comma18.,
    sum(frequency<0)                 as n_neg_freq format=comma18.,
    sum(gross_loss<0)                as n_neg_gross format=comma18.,
    sum(net_loss<0)                  as n_neg_net format=comma18.,
    sum(. = event_type)              as n_missing_event_type format=comma18.
  from aggregated_frequency;
quit; title;

/* 2) Reconcile quarterly table to event-level rollup */
title "Reconciliation: aggregated_frequency vs. cleaned_severity_data";
proc sql;
  /* Rebuild quarter × L1 totals from event-level */
  create table work._q_from_csd as
  select
    'Basel Event Type Level 1'n as event_type,
    intnx('qtr', Posting_Date, 0, 'e') as data_date format=date9.,
    count(distinct 'Internal Events'n) as frequency,
    sum(gross_loss) as gross_loss,
    sum(net_loss)  as net_loss
  from ops.cleaned_severity_data
  group by event_type, data_date;

  /* Compare to provided quarterly file */
  create table work._q_diff as
  select coalesce(a.event_type,b.event_type) as event_type length=80,
         coalesce(a.data_date,b.data_date)   as data_date format=date9.,
         a.frequency as freq_af, b.frequency as freq_csd,
         a.gross_loss as gl_af, b.gross_loss as gl_csd,
         a.net_loss   as nl_af, b.net_loss   as nl_csd,
         (a.frequency ne b.frequency
          or round(a.gross_loss,0.01) ne round(b.gross_loss,0.01)
          or round(a.net_loss,0.01)   ne round(b.net_loss,0.01)) as mismatch
  from aggregated_frequency a
  full join work._q_from_csd b
    on a.event_type=b.event_type and a.data_date=b.data_date;

  /* Summary + a small sample of mismatches */
  select sum(mismatch) as qtr_l1_mismatches from work._q_diff;
quit; title;

proc print data=work._q_diff(where=(mismatch=1) obs=&MAX_PRINT) noobs;
  title "Sample mismatches (if any)";
run; title;
