/*========================================
  Reconciliation (robust, alias both sides)
========================================*/
title "Reconciliation: aggregated_frequency vs. cleaned_severity_data";

proc sql;

/* 1) Normalize aggregated_frequency to use event_type alias */
create view work.af_norm as
select 
  'Basel Event Type Level 1'n as event_type length=80,
  data_date,
  frequency,
  gross_loss,
  net_loss
from aggregated_frequency;

/* 2) Rebuild quarter x L1 from event-level, using same alias */
create view work.csd_roll as
select
  'Basel Event Type Level 1'n as event_type length=80,
  intnx('qtr', Posting_Date, 0, 'e') as data_date format=date9.,
  count(distinct 'Internal Events'n) as frequency,
  sum(gross_loss) as gross_loss,
  sum(net_loss)  as net_loss
from ops.cleaned_severity_data
group by event_type, data_date;

/* 3) Compare */
create table work._q_diff as
select 
  coalesce(a.event_type, b.event_type) as event_type length=80,
  coalesce(a.data_date,  b.data_date ) as data_date   format=date9.,
  a.frequency as freq_af, b.frequency as freq_csd,
  a.gross_loss as gl_af,  b.gross_loss as gl_csd,
  a.net_loss   as nl_af,  b.net_loss   as nl_csd,
  ( a.frequency ne b.frequency
    or round(a.gross_loss,0.01) ne round(b.gross_loss,0.01)
    or round(a.net_loss, 0.01)  ne round(b.net_loss, 0.01) ) as mismatch
from work.af_norm a
full join work.csd_roll b
  on a.event_type = b.event_type
 and a.data_date  = b.data_date;

/* 4) Summary */
select sum(mismatch) as qtr_l1_mismatches
from work._q_diff;

quit;
title;

proc print data=work._q_diff(where=(mismatch=1) obs=20) noobs;
  title "Sample mismatches (if any)";
run;
title;
