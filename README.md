/* =========================
   Obligation + LGD-approval repeat logic
   Keys used:
     - obligation_key : STI_obl_nbr (fallback to c_obl if needed)
     - lgd_app_id     : fpb_lgdappid
   Material override = abs(OVRD_SEVERITY) > 1
   ========================= */

/* 1) Assemble a single table across quarters with the identifiers you need */
data _oblapp_all;
  length quarter $7 obligation_key $40 lgd_app_id $40;
  set
    ogm.OVERRIDE_2024Q2_DEDUP_ABL (in=q2)
    ogm.OVERRIDE_2024Q3_DEDUP_ABL (in=q3)
    ogm.OVERRIDE_2024Q4_DEDUP_ABL (in=q4)
    ogm.OVERRIDE_2025Q1_DEDUP_ABL (in=q1)
  ;

  /* Label the quarter flag */
  if q2 then quarter='2024Q2';
  else if q3 then quarter='2024Q3';
  else if q4 then quarter='2024Q4';
  else if q1 then quarter='2025Q1';

  /* Derive quarter index for proper time ordering */
  qtr_idx = input(substr(quarter,1,4),8.)*10 + input(substr(quarter,6,1),8.);

  /* Obligation + LGD approval composite keys (to character for stability) */
  obligation_key = coalescec(
                     strip(put(STI_obl_nbr, best16.)),
                     strip(put(c_obl,      best16.))
                   );
  lgd_app_id    = strip(put(fpb_lgdappid, best16.));  /* approval id */

  /* If OVRD_SEVERITY missing, rebuild from grades (matches your earlier logic) */
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num, model_lgd_grade_num)=0 then
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;

  /* Normalize override_ind if missing */
  if missing(override_ind) then override_ind = (OVRD_SEVERITY ne 0);

  /* Material = >1 notch in absolute value */
  material_1notch = (abs(OVRD_SEVERITY) > 1);

  keep quarter qtr_idx obligation_key lgd_app_id
       override_ind OVRD_SEVERITY material_1notch Override_Reason
       TFC_Curr_Bal tfc_face_amt;
run;

/* 2) Sort by the composite key then time */
proc sort data=_oblapp_all out=_oblapp_sorted;
  by obligation_key lgd_app_id qtr_idx;
run;

/* 3) Flag repeats for the SAME (obligation, approval) pair across ANY prior quarter */
data _oblapp_flags;
  set _oblapp_sorted;
  by obligation_key lgd_app_id;

  retain prior_mat 0;
  if first.lgd_app_id then prior_mat = 0;      /* reset at start of each (obligation, app) */

  repeat_material = 0;
  if material_1notch=1 then do;
    if prior_mat > 0 then repeat_material = 1; /* seen material earlier for this pair */
    prior_mat + 1;                              /* count this material for future rows */
  end;
run;

/* 4) Quarter summary (obligation+LGD level) */
proc sql;
  create table oblapp_material_repeat_by_qtr as
  select quarter,
         count(*)                                    as total_observations,
         sum(material_1notch)                        as mat_total,
         sum(repeat_material)                        as mat_repeat,
         sum(material_1notch) - sum(repeat_material) as mat_first_time,
         calculated mat_repeat    / max(calculated mat_total,1) as mat_repeat_pct format=percent8.2,
         calculated mat_first_time/ max(calculated mat_total,1) as mat_first_time_pct format=percent8.2
  from _oblapp_flags
  group by quarter
  order by quarter;
quit;

/* 5) (Optional) Top-10 reasons within each quarter at this level */
proc sql;
  create table oblapp_by_reason as
  select quarter,
         coalesce(Override_Reason,'(missing)') as reason length=200,
         count(*)                              as obs,
         sum(material_1notch)                  as mat_cnt,
         sum(material_1notch)/max(count(*),1)  as mat_rate format=percent8.2
  from _oblapp_flags
  group by quarter, calculated reason;
quit;

proc sort data=oblapp_by_reason out=oblapp_reason_sorted;
  by quarter descending mat_cnt reason;
run;

data oblapp_reason_top10;
  set oblapp_reason_sorted;
  by quarter;
  retain rank_within_qtr;
  if first.quarter then rank_within_qtr=0;
  rank_within_qtr+1;
  if rank_within_qtr<=10;
run;

/* Prints that mirror your existing outputs (now at obligation+LGD level) */
title "Material Overrides Repeat by Quarter (Obligation + LGD Approval)";
proc print data=oblapp_material_repeat_by_qtr noobs label;
  var quarter total_observations mat_total mat_repeat mat_repeat_pct mat_first_time mat_first_time_pct;
run;

title "Top 10 Material Override Reasons (by frequency within quarter, Obligation + LGD Approval)";
proc print data=oblapp_reason_top10 noobs label;
  var quarter reason mat_cnt mat_rate;
run;
title;
