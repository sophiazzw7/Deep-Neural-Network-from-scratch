/*--------------------------------------------
  Step 1: Basic summary by Event Type
  Missingness check + key statistics
---------------------------------------------*/
title "Summary Statistics and Missingness by Basel Event Type";

proc means data=ops.cleaned_severity_data n nmiss mean std min p1 p5 p25 p50 p75 p95 p99 max;
    class 'Basel Event Type Level 1'n;
    var gross_loss net_loss;
run;

/*--------------------------------------------
  Step 2: Outlier detection using interquartile range
  Flag events where gross_loss > Q3 + 1.5 * IQR
---------------------------------------------*/

proc sort data=ops.cleaned_severity_data out=sorted_sev;
    by 'Basel Event Type Level 1'n;
run;

proc univariate data=sorted_sev noprint;
    by 'Basel Event Type Level 1'n;
    var gross_loss;
    output out=sev_stats pctlpre=P_ pctlpts=25,75;
run;

data sev_iqr_flagged;
    merge sorted_sev sev_stats;
    by 'Basel Event Type Level 1'n;
    iqr = P_75 - P_25;
    upper_limit = P_75 + 1.5 * iqr;
    lower_limit = P_25 - 1.5 * iqr;
    if gross_loss > upper_limit then outlier_flag = 1;
    else if gross_loss < lower_limit then outlier_flag = -1;
    else outlier_flag = 0;
run;

/*--------------------------------------------
  Step 3: Outlier summary by Event Type
---------------------------------------------*/
title "Outlier Summary by Basel Event Type Level 1";

proc sql;
    create table outlier_summary as
    select 
        'Basel Event Type Level 1'n as event_type,
        count(*) as total_records,
        sum(outlier_flag=1) as high_outliers,
        sum(outlier_flag=-1) as low_outliers,
        calculated high_outliers / calculated total_records as pct_high format=percent8.2
    from sev_iqr_flagged
    group by 'Basel Event Type Level 1'n;
quit;

proc print data=outlier_summary noobs label;
    label event_type = 'Basel Event Type Level 1'
          total_records = 'Total Records'
          high_outliers = 'High Gross Loss Outliers'
          pct_high = '% High Outliers';
run;

/*--------------------------------------------
  Step 4: Optional – review of top outlier records
---------------------------------------------*/
title "Top Gross Loss Outliers by Basel Event Type";

proc sort data=sev_iqr_flagged out=top_outliers;
    where outlier_flag=1;
    by 'Basel Event Type Level 1'n descending gross_loss;
run;

proc print data=top_outliers(obs=20);
    var 'Basel Event Type Level 1'n 'Internal Events'n Posting_Date gross_loss net_loss;
run;
title;
