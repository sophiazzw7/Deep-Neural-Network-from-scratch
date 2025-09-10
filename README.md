/* ========= MOD1497 override repeat analysis (NO MACROS) ========= */
/* 1) Point to where you saved the developer DEDUP tables */
options compress=yes reuse=yes;
libname ogm "/sasdata/mrmg1/MOD1497/override_dedup" access=readonly;
/* Optional place to save outputs; comment out if you only want WORK */
libname out "/sasdata/mrmg1/MOD1497/override_results";

/* 2) Stack ABL across quarters (edit the list if you have different quarters) */
data _ABL;
  length quarter $7 seg $3 obligor_key $64;
  set
    ogm.OVERRIDE_2024Q2_DEDUP_ABL (in=in1  keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                                                 model_lgd_grade_num final_lgd_grade_num Override_Reason)
    ogm.OVERRIDE_2024Q3_DEDUP_ABL (in=in2  keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                                                 model_lgd_grade_num final_lgd_grade_num Override_Reason)
    ogm.OVERRIDE_2024Q4_DEDUP_ABL (in=in3  keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                                                 model_lgd_grade_num final_lgd_grade_num Override_Reason)
    ogm.OVERRIDE_2025Q1_DEDUP_ABL (in=in4  keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                                                 model_lgd_grade_num final_lgd_grade_num Override_Reason)
  ;
  seg = "ABL";
  obligor_key = c_obgobl;

  if in1 then quarter = "2024Q2";
  else if in2 then quarter = "2024Q3";
  else if in3 then quarter = "2024Q4";
  else if in4 then quarter = "2025Q1";

  /* Recompute/patch if needed */
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0 then
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;

  if missing(override_ind) and not missing(OVRD_SEVERITY) then
    override_ind = (OVRD_SEVERITY ne 0);

  /* Developer definition of material override */
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

/* 3) Stack SF across quarters */
data _SF;
  length quarter $7 seg $3 obligor_key $64;
  set
    ogm.OVERRIDE_2024Q2_DEDUP_SF (in=in1  keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                                             model_lgd_grade_num final_lgd_grade_num Override_Reason)
    ogm.OVERRIDE_2024Q3_DEDUP_SF (in=in2  keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                                             model_lgd_grade_num final_lgd_grade_num Override_Reason)
    ogm.OVERRIDE_2024Q4_DEDUP_SF (in=in3  keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                                             model_lgd_grade_num final_lgd_grade_num Override_Reason)
    ogm.OVERRIDE_2025Q1_DEDUP_SF (in=in4  keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                                             model_lgd_grade_num final_lgd_grade_num Override_Reason)
  ;
  seg = "SF";
  obligor_key = c_obgobl;

  if in1 then quarter = "2024Q2";
  else if in2 then quarter = "2024Q3";
  else if in3 then quarter = "2024Q4";
  else if in4 then quarter = "2025Q1";

  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0 then
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;

  if missing(override_ind) and not missing(OVRD_SEVERITY) then
    override_ind = (OVRD_SEVERITY ne 0);

  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

/* 4) Combine ABL + SF */
data allq;
  set _ABL _SF;
run;

/* 5) Summary by quarter × segment */
proc sql;
  create table ovr_summary_by_qtr_seg as
  select quarter, seg,
         count(*)                                    as obs_cnt,
         sum(override_ind)                            as ovr_cnt,
         calculated ovr_cnt / calculated obs_cnt      as ovr_rate format=percent8.2,
         sum(material_1notch)                         as mat_cnt,
         calculated mat_cnt / calculated obs_cnt      as mat_rate format=percent8.2
  from allq
  group by quarter, seg
  order by quarter, seg;
quit;

/* 6) Repeats across quarters (obligor-level) */
proc sql;
  create table _per_obligor as
  select seg, obligor_key,
         count(distinct quarter) as qtrs_seen,
         sum(override_ind>0)    as qtrs_overridden,
         sum(material_1notch>0) as qtrs_material,
         min(quarter)           as first_qtr,
         max(quarter)           as last_qtr
  from allq
  group by seg, obligor_key;
quit;

data repeat_overrides_any;      set _per_obligor; if qtrs_overridden >= 2; run;
data repeat_overrides_material; set _per_obligor; if qtrs_material  >= 2; run;

/* 7) (Optional) Save results to a permanent library */
proc datasets lib=work nolist;
  copy in=work out=out memtype=data;
  select ovr_summary_by_qtr_seg _per_obligor repeat_overrides_any repeat_overrides_material allq;
quit;

/* 8) Quick looks (optional) */
title "Override & Material Rates by Quarter × Segment";
proc print data=ovr_summary_by_qtr_seg noobs; run;

title "Obligors with Repeated Overrides (≥2 quarters)";
proc print data=repeat_overrides_any(obs=100) noobs; run;

title "Obligors with Repeated MATERIAL Overrides (≥2 quarters)";
proc print data=repeat_overrides_material(obs=100) noobs; run;
title;
