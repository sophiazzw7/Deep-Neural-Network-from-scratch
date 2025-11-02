/* Check if any event’s GL lines sum to zero or flip sign */
proc sql;
create table event_sign_check as
select 'Internal Events'n,
sum('GL Amount'n) as total_amt,
min('GL Amount'n) as min_line,
max('GL Amount'n) as max_line
from ops.operational_risk_loss_forecast
group by 'Internal Events'n
having (total_amt = 0 or min_line < 0 and max_line > 0);
quit;


proc freq data=ops.operational_risk_loss_forecast;
  tables 'Event Record Type'n * 'Type of Impact'n / missing norow nocol nopercent;
run;

data date_diff_check;
  set ops.operational_risk_loss_forecast;
  diff_days = 'Posting Date'n - 'Event Start Date'n;
  if diff_days < 0 or diff_days > 3650; /* >10 years or negative */
run;

proc means data=ops.operational_risk_loss_forecast n min mean max;
  var 'GL Amount'n 'Recovery Amount'n 'Gross Loss Amount'n;
run;

proc sort data=ops.operational_risk_loss_forecast out=_dupchk dupout=_true_dups nodupkey;
  by 'Internal Events'n 'GL Tracking ID'n 'GL Amount'n 'Posting Date'n;
run;

proc sql; select count(*) as n_true_dups from _true_dups; quit;

proc sql;
  select count(*) as n_negative_events
  from work.cleaned_severity_data
  where net_loss < 0;
quit;

proc freq data=ops.operational_risk_loss_forecast;
  tables 'Basel Event Type Level 1'n * 'Basel Event Type Level 2'n / missing;
run;
