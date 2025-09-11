/* ================================
   ABL — Build panel & repeat flags
   Key for persistence: c_obg (OBLIGOR)
   Material override: abs(OVRD_SEVERITY) > 1
   ================================ */

/* -------- 1) Stack quarter-level override extracts (ABL only) -------- */
/* Keep exactly the developer-used fields; do NOT change exclusions/defs */
data _ABL_q2 _ABL_q3 _ABL_q4 _ABL_q1;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  /* ---------- 2024Q2 ---------- */
  set ogm.OVERRIDE_2024Q2_DEDUP_ABL
    (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
          model_lgd_grade_num final_lgd_grade_num);
  seg      = 'ABL';
  quarter  = '2024Q2';
  obligor_key = c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY) then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

data _ABL_q3;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  set ogm.OVERRIDE_2024Q3_DEDUP_ABL
    (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
          model_lgd_grade_num final_lgd_grade_num);
  seg='ABL'; quarter='2024Q3'; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY) then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

data _ABL_q4;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  set ogm.OVERRIDE_2024Q4_DEDUP_ABL
    (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
          model_lgd_grade_num final_lgd_grade_num);
  seg='ABL'; quarter='2024Q4'; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY) then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

data _ABL_q1;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  set ogm.OVERRIDE_2025Q1_DEDUP_ABL
    (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
          model_lgd_grade_num final_lgd_grade_num);
  seg='ABL'; quarter='2025Q1'; obligor_key=c_obgobl;
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;
  if missing(override_ind) and not missing(OVRD_SEVERITY) then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

/* Union all quarters */
data _ABL;
  set _ABL_q2 _ABL_q3 _ABL_q4 _ABL_q1;
run;

proc datasets lib=work nolist; delete _ABL_q2 _ABL_q3 _ABL_q4 _ABL_q1; quit;

/* -------- 2) Bring in OBLIGOR key (c_obg) from attributes -------- */
/* Build an attributes table with stable widths and quarter labels.   */
/* This assumes each quarterly *_DEDUP_ABL contains c_obg + c_obgobl */
data abl_attrs;
  length quarter $7 c_obg $200 c_obgobl $200;
  set
    ogm.OVERRIDE_2024Q2_DEDUP_ABL (in=i2 keep=c_obg c_obgobl)
    ogm.OVERRIDE_2024Q3_DEDUP_ABL (in=i3 keep=c_obg c_obgobl)
    ogm.OVERRIDE_2024Q4_DEDUP_ABL (in=i4 keep=c_obg c_obgobl)
    ogm.OVERRIDE_2025Q1_DEDUP_ABL (in=i5 keep=c_obg c_obgobl)
  ;
  if      i2 then quarter='2024Q2';
  else if i3 then quarter='2024Q3';
  else if i4 then quarter='2024Q4';
  else if i5 then quarter='2025Q1';
run;

proc sort data=abl_attrs nodupkey; by quarter c_obgobl; run;

/* Left join to carry c_obg (OBLIGOR) onto the working panel */
proc sql;
  create table _ABL_enriched as
  select a.*, b.c_obg
  from _ABL as a
  left join abl_attrs as b
    on a.quarter=b.quarter and a.c_obgobl=b.c_obgobl;
quit;

/* Sanity checks in the log */
proc sql noprint;
  select count(*) into :_miss_obg trimmed
  from _ABL_enriched where missing(c_obg);
quit;
%put NOTE: ABL repeat logic — observations with missing c_obg after join = &_miss_obg ;
%if %sysevalf(&_miss_obg > 0) %then %do;
  %put WARNING: Some rows lack c_obg (obligor id). Repeat flags will skip those rows.;
%end;

/* -------- 3) Sort by obligor and time; build repeat flags -------- */
data _ABL_indexed;
  set _ABL_enriched;
  /* sortable quarter index: YYYY*10 + Q */
  qtr_idx = input(substr(quarter,1,4),8.)*10 + input(substr(quarter,6,1),8.);
run;

proc sort data=_ABL_indexed out=_ABL_sorted;
  by c_obg qtr_idx c_obgobl;
run;

/* Persist across *quarters* at the OBLIGOR level (c_obg) */
data ABL_flags;
  set _ABL_sorted;
  by c_obg qtr_idx;

  length repeat_material repeat_any 8 prev_qtr $7;
  retain prior_mat prior_any prev_qtr;

  if first.c_obg then do;           /* reset per obligor */
    prior_mat = 0;
    prior_any = 0;
    prev_qtr  = '';
  end;

  /* repeat if there has been *any* prior quarter with the condition */
  repeat_material = (material_1notch=1 and prior_mat>0);
  repeat_any      = (override_ind=1     and prior_any>0);

  /* update running state after evaluating repeat flags */
  if material_1notch=1 then prior_mat + 1;
  if override_ind=1     then prior_any + 1;
  prev_qtr = quarter;
run;

/* -------- 4) Quarter-level totals for QA & reporting -------- */
proc sql;
  create table abl_material_repeat_by_qtr as
  select
      quarter,
      count(*)                                    as total_observations,
      sum(material_1notch)                        as mat_total,
      sum(repeat_material)                        as mat_repeat,
      /* First-time material = material minus repeated */
      calculated mat_total - calculated mat_repeat as mat_first_time,
      /* Shares */
      calculated mat_repeat     / max(calculated mat_total,1) format=percent8.2 as mat_repeat_pct,
      calculated mat_first_time / max(calculated mat_total,1) format=percent8.2 as mat_first_time_pct
  from ABL_flags
  /* skip rows without obligor id (cannot compute persistence) */
  where not missing(c_obg)
  group by quarter
  order by quarter;
quit;

/* Optional: distinct-obligor view (helps QA the key choice) */
proc sql;
  create table abl_obligor_level as
  select quarter,
         count(distinct case when material_1notch=1 then c_obg end) as mat_obligors,
         count(distinct c_obg)                                      as total_obligors
  from ABL_flags
  where not missing(c_obg)
  group by quarter
  order by quarter;
quit;

/* Clean up temps; keep ABL_flags + two summary tables */
proc datasets lib=work nolist;
  delete _ABL _ABL_enriched _ABL_indexed _ABL_sorted abl_attrs;
quit;
