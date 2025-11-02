/*=====================*
 * Simple Quick Health KPI
 *=====================*/
proc sql noprint;
  select count(*) into :n_rows from cleaned_data;
  select count(*) into :n_dups from work._dups_;
  select sum(flag_unreasonable) into :n_flagged from work._checks_;
  select count(*) into :n_lt10k from work._gross_loss where Gross_Loss < 10000;
  select sum(missing('Posting Date'n)) into :n_miss_date from cleaned_data;
  select sum(missing(GL_Amount)) into :n_miss_gl from cleaned_data;
quit;

data kpi_summary;
  length Metric $40 Value 8;
  Metric="Total rows"; Value=&n_rows; output;
  Metric="Duplicate events"; Value=&n_dups; output;
  Metric="Flagged rows"; Value=&n_flagged; output;
  Metric="Missing Posting Date"; Value=&n_miss_date; output;
  Metric="Missing GL_Amount"; Value=&n_miss_gl; output;
  Metric="Gross_Loss < 10K events"; Value=&n_lt10k; output;
run;

title "Quick Health KPI Summary";
proc print data=kpi_summary noobs label;
  label Metric="Check Item" Value="Count";
run;
title;

