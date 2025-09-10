/* Point to your saved DEDUP tables */
libname ogm "/sasdata/mrmg1/MOD1497/override_dedup" access=readonly;

/* 1) Empty shell with WIDE lengths (prevents truncation) */
data _ABL;
  length quarter $7 seg $3;
  length c_obgobl $200 obligor_key $200;     /* <- widen to 200 */
  length f_uplddt 8 override_ind 8 OVRD_SEVERITY 8 
         model_lgd_grade_num 8 final_lgd_grade_num 8 material_1notch 8;
  stop;
run;

/* --- Helper snippet: append one quarter (copy/paste for each) --- */
/* 2024Q2 */
data _ABL_q2;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  set ogm.OVERRIDE_2024Q2_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg="ABL"; quarter="2024Q2"; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY)
    then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;
proc append base=_ABL data=_ABL_q2 force; run;
proc datasets lib=work nolist; delete _ABL_q2; quit;

/* 2024Q3 */
data _ABL_q3;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  set ogm.OVERRIDE_2024Q3_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg="ABL"; quarter="2024Q3"; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY)
    then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;
proc append base=_ABL data=_ABL_q3 force; run;
proc datasets lib=work nolist; delete _ABL_q3; quit;

/* 2024Q4 */
data _ABL_q4;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  set ogm.OVERRIDE_2024Q4_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg="ABL"; quarter="2024Q4"; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY)
    then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;
proc append base=_ABL data=_ABL_q4 force; run;
proc datasets lib=work nolist; delete _ABL_q4; quit();

/* 2025Q1 */
data _ABL_q1;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  set ogm.OVERRIDE_2025Q1_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg="ABL"; quarter="2025Q1"; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY)
    then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;
proc append base=_ABL data=_ABL_q1 force; run;
proc datasets lib=work nolist; delete _ABL_q1; quit;
Summaries + repeaters (no PRINTs to dodge EG viewer)
sas
Copy code
/* Rates by quarter (ABL) */
proc sql;
  create table ovr_summary_by_qtr_ABL as
  select quarter,
         count(*) as obs_cnt,
         sum(override_ind) as ovr_cnt,
         calculated ovr_cnt / calculated obs_cnt as ovr_rate format=percent8.2,
         sum(material_1notch) as mat_cnt,
         calculated mat_cnt / calculated obs_cnt as mat_rate format=percent8.2
  from _ABL
  group by quarter
  order by quarter;
quit;

/* Repeats across quarters (obligor-level) */
proc sql;
  create table _per_obligor_ABL as
  select obligor_key,
         count(distinct quarter) as qtrs_seen,
         sum(override_ind>0)    as qtrs_overridden,
         sum(material_1notch>0) as qtrs_material,
         min(quarter) as first_qtr,
         max(quarter) as last_qtr
  from _ABL
  group by obligor_key;
quit;

data repeat_overrides_any_ABL;      set _per_obligor_ABL; if qtrs_overridden >= 2; run;
data repeat_overrides_material_ABL; set _per_obligor_ABL; if qtrs_material  >= 2; run;
(Optional) Peek without heavy output
sas
Copy code
/* Show a tiny sample, avoids big ODS rendering */
proc sql outobs=10;
  select * from ovr_summary_by_qtr_ABL;
  select * from repeat_overrides_material_ABL;
quit;
(Optional) Persist results later
sas
Copy code
libname out "/sasdata/mrmg1/MOD1497/override_results";
proc datasets lib=work nolist;
  copy in=work out=out memtype=data;
  select _ABL ovr_summary_by_qtr_ABL _per_obligor_ABL
         repeat_overrides_any_ABL repeat_overrides_material_ABL;
quit;
Why this fixes it

All c_obgobl/obligor_key lengths are $200 end-to-end → no truncation warnings.

No PROC PRINT (common EG trigger for that submit error). You still get the datasets; use the small PROC SQL outobs=10 to sanity-check.

If you still get the submit error even on the tiny outobs=10 query, disconnect/reconnect the EG server session and try just the last proc sql outobs=10; block alone to verify the connection.
