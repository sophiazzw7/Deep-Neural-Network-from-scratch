proc sql;
  create table zero_net_summary as
  select 
    'Basel Event Type Level 1'n as event_type,
    count(*) as n_zero_net,
    count(*) / (select count(*) from ops.cleaned_severity_data) as pct_total format=percent8.2
  from ops.cleaned_severity_data
  where net_loss = 0
  group by event_type;
quit;

proc print data=zero_net_summary noobs; run;
→ This will let you see how many fully recovered (or rounded to zero) events remain by ET. If any ET has >5% zeros, that’s a flag.

(b) Top Quarters by Frequency and Net Loss
sas
Copy code
proc sql;
  create table top_quarters as
  select 
    'Basel Event Type Level 1'n as event_type,
    data_date,
    frequency,
    net_loss,
    gross_loss
  from aggregated_frequency
  group by event_type, data_date
  order by net_loss desc;
quit;

proc sort data=top_quarters out=top5_per_et;
  by event_type descending net_loss;
run;

proc print data=top5_per_et(obs=35) noobs;
  title "Top 5 Quarters by Net Loss per Event Type";
run;
