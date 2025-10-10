gotcha—let’s QC **only what PSI actually uses** from `f_app_score_flat` for the **current** perf year:

**Vars to check (from your PSI code):**
`score_value, eff_from_dt, score_component, bdfs_blended, vintage_yr (or YEAR(eff_from_dt)), sample_tp`
(+ I’ll also peek at `weight`, since your earlier plots raised a question.)

---

### SAS – compact QC script (drop-in)

```sas
/*-- point libname to where your file lives --*/
libname mydata "<your path>";

/*===== 1) Make the CURRENT slice for the latest perf year =====*/
data _flat_base;
  set mydata.f_app_score_flat;
  vintage_yr_calc = year(eff_from_dt);        /* in case vintage_yr is missing */
run;

proc sql noprint;
  /* match your PSI logic: ignore the 2015 “INTIME” marker date */
  select max(vintage_yr_calc) into :perfyr
  from _flat_base
  where eff_from_dt ne '30JUN2015'd and upcase(sample_tp)='CURRENT';
quit;

%put NOTE: PERF YEAR =&perfyr.;

data flat_current;
  set _flat_base;
  where upcase(sample_tp)='CURRENT' and vintage_yr_calc=&perfyr;
run;

/*===== 2) Basic shape =====*/
title "Flat(Current) — counts & time window";
proc sql;
  select count(*) as n_rows
       ,count(distinct unique_id) as n_unique_id
       ,min(eff_from_dt) format=date9. as min_eff_from
       ,max(eff_from_dt) format=date9. as max_eff_from
  from flat_current;
quit;

/*===== 3) Missingness & ranges for PSI vars (+weight) =====*/
title "Missingness and ranges (PSI vars)";
proc means data=flat_current n nmiss min p1 p5 q1 median mean q3 p95 p99 max maxdec=2;
  var score_value weight;
run;

proc freq data=flat_current;
  tables score_component bdfs_blended / missing;
run;

/* Month-by-month volume within perf year (sanity) */
proc sql;
  title "Volume by eff_from_dt month (within &perfyr)";
  select put(eff_from_dt, yymmn6.) as yyyymm, count(*) as n
  from flat_current
  group by calculated yyyymm
  order by calculated yyyymm;
quit;

/*===== 4) Specific validity checks =====*/
title "Potential Issues (invalid/out-of-range)";
data flat_issues;
  set flat_current;
  length issue $60;
  if missing(score_value)                      then issue='Missing score';
  else if score_value < 0                      then issue='Negative score';
  else if score_value > 1000                   then issue='Score > 1000 (check scaling)';
  if not missing(weight) and weight <= 0       then issue=coalescec(issue,'')||'| Non-positive weight';
  if score_component not in (1,2,.)            then issue=coalescec(issue,'')||'| Unexpected score_component';
  if upcase(bdfs_blended) not contains 'SCORE' then issue=coalescec(issue,'')||'| Unexpected bdfs_blended';
  if year(eff_from_dt) ne &perfyr              then issue=coalescec(issue,'')||'| eff_from_dt outside perf year';
  if not missing(issue) then output;
run;

proc sql;
  select issue, count(*) as n
  from flat_issues
  group by issue
  order by n desc;
quit;

proc print data=flat_issues (obs=25);
  title "Sample offending rows";
run;

/*===== 5) Duplicate logic =====*/
/* Business key for PSI rows should be (unique_id, eff_from_dt).
   Multiple months per unique_id across the year are OK; duplicates WITHIN the same month are not. */
proc sql;
  create table dup_keys as
  select unique_id, eff_from_dt, count(*) as n_rows
  from flat_current
  group by unique_id, eff_from_dt
  having count(*)>1;
quit;

title "Exact duplicates within (unique_id, eff_from_dt)";
proc sql; select count(*) as dup_rows from dup_keys; quit;

proc print data=dup_keys(obs=25); run;

/* If you also want to see whether same (unique_id, eff_from_dt) has different scores */
proc sql;
  create table dup_diffscore as
  select unique_id, eff_from_dt
  from flat_current
  group by unique_id, eff_from_dt
  having min(score_value) ne max(score_value);
quit;

title "Same (unique_id,eff_from_dt) but differing score_value";
proc sql; select count(*) as n_problem from dup_diffscore; quit;
proc print data=dup_diffscore(obs=25); run;

/*===== 6) The 15.478 weight spike check (CURRENT only) =====*/
title "Weight spike check (15.478) in CURRENT &perfyr";
proc sql;
  select count(*) as n_equal_15478
       ,min(weight) as min_w, max(weight) as max_w
  from flat_current
  where round(weight,1e-3)=15.478;   /* guard for floating precision */
quit;

/* Quick distribution plots (optional) */
ods graphics on;
title "Score distribution — CURRENT &perfyr";
proc sgplot data=flat_current;
  histogram score_value / nbins=40;
  density score_value;
  xaxis label="score_value";
run;

title "Weight distribution — CURRENT &perfyr";
proc sgplot data=flat_current;
  histogram weight;
  density weight;
  xaxis label="weight";
run;
ods graphics off;
title;
```

---

### How this lines up with the ETL/PSI logic

* **Perf year**: mirrors your PSI code—`&perfyr` is the **max** year of `eff_from_dt` with `sample_tp='CURRENT'` (ignoring the 2015 marker date).
* **Duplicates**: since PSI groups by month, the right uniqueness test is **(unique_id, eff_from_dt)**. Multiple months per id in the year are expected; >1 rows for the *same month* are not.
* **Score validity**: your earlier finding of negative scores in the INTIME sample is flagged here too if any appear in CURRENT.
* **Weight 15.478**: this explicitly counts how many CURRENT rows equal that value so you can confirm whether that spike exists in the PSI population. (In some builds, weight comes from upstream sampling/expansion—if you see a spike at 15.478 we can ask the dev whether that’s an intended expansion factor.)

If you want, I can add a short **recon** between `flat_current` and `cubes.bdfs_am_final_summary_202412` for 2023 start dates to prove consistency (e.g., counts by month, overlap on `unique_id+eff_from_dt`).
