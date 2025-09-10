

/* Create an empty shell so APPEND has a target */
data _ABL;
  length quarter $7 seg $3 obligor_key $64;
  length c_obgobl $64;  /* keep the raw key too if you like */
  length f_uplddt 8 override_ind 8 OVRD_SEVERITY 8 
         model_lgd_grade_num 8 final_lgd_grade_num 8 material_1notch 8;
  stop;
run;

/* ---- 2024Q2 ABL ---- */
data _ABL_q2;
  length quarter $7 seg $3 obligor_key $64 material_1notch 8;
  set ogm.OVERRIDE_2024Q2_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg       = "ABL";
  quarter   = "2024Q2";
  obligor_key = c_obgobl;

  /* Patch severity if missing; then material flag */
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0 then
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY) then
    override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

proc append base=_ABL data=_ABL_q2 force; run; proc datasets lib=work nolist; delete _ABL_q2; quit;

/* ---- 2024Q3 ABL ---- */
data _ABL_q3;
  length quarter $7 seg $3 obligor_key $64 material_1notch 8;
  set ogm.OVERRIDE_2024Q3_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg="ABL"; quarter="2024Q3"; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0 then
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY) then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

proc append base=_ABL data=_ABL_q3 force; run; proc datasets lib=work nolist; delete _ABL_q3; quit;

/* ---- 2024Q4 ABL ---- */
data _ABL_q4;
  length quarter $7 seg $3 obligor_key $64 material_1notch 8;
  set ogm.OVERRIDE_2024Q4_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg="ABL"; quarter="2024Q4"; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0 then
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY) then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

proc append base=_ABL data=_ABL_q4 force; run; proc datasets lib=work nolist; delete _ABL_q4; quit;

/* ---- 2025Q1 ABL ---- */
data _ABL_q1;
  length quarter $7 seg $3 obligor_key $64 material_1notch 8;
  set ogm.OVERRIDE_2025Q1_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg="ABL"; quarter="2025Q1"; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0 then
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY) then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

proc append base=_ABL data=_ABL_q1 force; run; proc datasets lib=work nolist; delete _ABL_q1; quit;

data allq; set _ABL _SF; run;

proc sql;
  create table ovr_summary_by_qtr_seg as
  select quarter, seg,
         count(*) as obs_cnt,
         sum(override_ind) as ovr_cnt,
         calculated ovr_cnt / calculated obs_cnt as ovr_rate format=percent8.2,
         sum(material_1notch) as mat_cnt,
         calculated mat_cnt / calculated obs_cnt as mat_rate format=percent8.2
  from allq
  group by quarter, seg
  order by quarter, seg;
quit;

proc sql;
  create table _per_obligor as
  select seg, obligor_key,
         count(distinct quarter) as qtrs_seen,
         sum(override_ind>0)    as qtrs_overridden,
         sum(material_1notch>0) as qtrs_material,
         min(quarter) as first_qtr,
         max(quarter) as last_qtr
  from allq
  group by seg, obligor_key;
quit;

data repeat_overrides_any;      set _per_obligor; if qtrs_overridden >= 2; run;
data repeat_overrides_material; set _per_obligor; if qtrs_material  >= 2; run;

title "Override & Material Rates by Quarter × Segment";
proc print data=ovr_summary_by_qtr_seg noobs; run;

title "Obligors with Repeated Overrides (≥2 quarters)";
proc print data=repeat_overrides_any(obs=100) noobs; run;

title "Obligors with Repeated MATERIAL Overrides (≥2 quarters)";
proc print data=repeat_overrides_material(obs=100) noobs; run;
title;

