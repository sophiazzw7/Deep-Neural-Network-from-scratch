/* ================================================================
   AH4950 – OGM 202412 Dataset Quick Review
   Purpose: basic structure, distribution, and data reasonableness
   ================================================================ */

/* 1. Assign a libname and short nickname */
libname mydata "C:\path\to\your\folder";   /* <-- change to your path */
data ogm;
    set mydata.bdfs_am_final_summary_202412;
run;

/* ---------------------------------------------------------------
   STEP 1 – Structure overview
   Checks: variable names, formats, and types
   --------------------------------------------------------------- */
title "Dataset Structure: bdfs_am_final_summary_202412";
proc contents data=ogm; run;

/* ---------------------------------------------------------------
   STEP 2 – Record count and unique IDs
   Checks: total observations, unique company_id and unique_id
   --------------------------------------------------------------- */
title "Basic Record Counts and Unique IDs";
proc sql;
  select count(*) as Total_Records format=comma10.,
         count(distinct company_id) as Unique_Companies format=comma10.,
         count(distinct unique_id)  as Unique_Records format=comma10.
  from ogm;
quit;

/* ---------------------------------------------------------------
   STEP 3 – Sample composition
   Variable: sample_tp (CURRENT vs INTIME)
   Purpose: check sample balance across vintages
   --------------------------------------------------------------- */
title "Sample Composition by Type and Vintage Year";
proc freq data=ogm;
  tables sample_tp*vintage_yr / norow nocol nocum nopercent;
run;

/* ---------------------------------------------------------------
   STEP 4 – Score & Weight Distribution
   Variables: score_value, weight
   Purpose: detect range, outliers, or abnormal scaling
   --------------------------------------------------------------- */
title "Score and Weight Distribution Summary Statistics";
proc means data=ogm n nmiss min p1 p5 q1 median mean q3 p95 p99 max std;
  var score_value weight;
run;

title "Histogram: Model Score (score_value)";
proc sgplot data=ogm;
  histogram score_value / nbins=40 fillattrs=(color=steelblue);
  density score_value;
  xaxis label="Model Score (score_value)";
  yaxis label="Frequency";
run;

title "Histogram: Record Weight (weight)";
proc sgplot data=ogm;
  histogram weight / nbins=40 fillattrs=(color=lightgray);
  density weight;
  xaxis label="Record Weight";
  yaxis label="Frequency";
run;

/* ---------------------------------------------------------------
   STEP 5 – Event rates (bad proportions)
   Variables: BAD_AT_3MO / 6MO / 12MO / EVER_BAD
   Purpose: understand portfolio risk levels
   --------------------------------------------------------------- */
title "Event Rates by Sample Type and Vintage Year";
proc sql;
  select sample_tp, vintage_yr,
         mean(BAD_AT_3MO)  as Bad_3mo format=percent8.2,
         mean(BAD_AT_6MO)  as Bad_6mo format=percent8.2,
         mean(BAD_AT_12MO) as Bad_12mo format=percent8.2,
         mean(EVER_BAD)    as Ever_Bad format=percent8.2
  from ogm
  group by sample_tp, vintage_yr;
quit;

/* ---------------------------------------------------------------
   STEP 6 – Weight sanity & duplicate key check
   Variables: weight, unique_id
   Purpose: ensure positive weights and unique records
   --------------------------------------------------------------- */
title "Weight Range and Missingness";
proc means data=ogm min max mean nmiss;
  var weight;
run;

title "Duplicate Record Check (sample_tp, vintage_yr, unique_id)";
proc sort data=ogm nodupkey dupout=dq_dups;
  by sample_tp vintage_yr unique_id;
run;

proc sql;
  select count(*) as Duplicate_Records from dq_dups;
quit;

/* ---------------------------------------------------------------
   STEP 7 – BAD variable consistency
   Variables: BAD_AT_0MO–BAD_AT_12MO, EVER_BAD
   Purpose: verify logical progression (monotonic increase)
   --------------------------------------------------------------- */
title "Correlation Among BAD Flags (should increase by horizon)";
proc corr data=ogm plots=matrix(histogram);
  var BAD_AT_0MO BAD_AT_3MO BAD_AT_6MO BAD_AT_9MO BAD_AT_12MO EVER_BAD;
run;

/* ---------------------------------------------------------------
   STEP 8 – LOB and sample coverage
   Variable: lob_indicator
   Purpose: ensure all LOBs represented appropriately
   --------------------------------------------------------------- */
title "Line of Business vs Sample Type Coverage";
proc freq data=ogm;
  tables lob_indicator*sample_tp / missing;
run;

/* ---------------------------------------------------------------
   STEP 9 – Time sanity checks
   Variables: start_date, rpt_prd_date_id
   Purpose: ensure booking vs reporting timeline consistency
   --------------------------------------------------------------- */
title "Time Range Check (Start Date vs Reporting Date)";
proc sql;
  select min(start_date) as Earliest_Start format=date9.,
         max(start_date) as Latest_Start format=date9.,
         min(rpt_prd_date_id) as Earliest_Rpt,
         max(rpt_prd_date_id) as Latest_Rpt
  from ogm;
quit;

/* ---------------------------------------------------------------
   STEP 10 – Quick data issue flags
   Purpose: find negative or invalid values
   --------------------------------------------------------------- */
title "Potential Data Issues (Invalid or Out-of-Range Values)";
data dq_issues;
  set ogm;
  length issue $50;
  if score_value < 0        then issue='Negative Score';
  if score_value > 1000     then issue='Score > 1000';
  if weight <= 0            then issue='Nonpositive Weight';
  if BAD_AT_12MO not in (0,1,.) then issue='Invalid BAD Flag';
  if EVER_BAD not in (0,1,.)   then issue='Invalid EVER_BAD';
  if not missing(issue) then output;
run;

proc print data=dq_issues(obs=20);
  var sample_tp vintage_yr unique_id score_value weight issue;
run;

title "Histogram: Model Score with Density Overlay";
proc sgplot data=ogm;
  histogram score_value / nbins=30 fillattrs=(color=cx6699CC);
  density score_value / lineattrs=(color=black);
  xaxis label="Score Value";
  yaxis label="Frequency";
  inset "Checking score_value distribution for scaling and outliers" / position=topright;
run;

title "Histogram: BAD_AT_12MO (0/1 Event Flag)";
proc sgplot data=ogm;
  vbar BAD_AT_12MO / datalabel fillattrs=(color=cxCC6666);
  xaxis label="BAD_AT_12MO Value";
  yaxis label="Count";
  inset "Checking binary event flag distribution" / position=topright;
run;

title "Histogram: Weight Variable";
proc sgplot data=ogm;
  histogram weight / nbins=30 fillattrs=(color=cxCCCC66);
  density weight;
  xaxis label="Record Weight";
  yaxis label="Frequency";
  inset "Checking weight reasonableness (should be positive)" / position=topright;
run;

/* ================================================================ */
title; footnote;
