/* 3) Robust quarter ordering and repeat flags (ABL) */

/* Create a numeric index so sort is truly chronological (e.g., 2024Q2 -> 20242) */
data _ABL;
  set _ABL;
  qtr_idx = input(substr(quarter,1,4),8.)*10 + input(substr(quarter,6,1),8.);
run;

proc sort data=_ABL out=_ABL_sorted;
  by obligor_key qtr_idx;
run;

data _ABL_flags;
  set _ABL_sorted;
  by obligor_key qtr_idx;

  length prev_qtr $7;                    /* type/length here */
  retain prior_mat prior_any prev_qtr;   /* retain state across rows of same obligor */

  /* reset running state per obligor */
  if first.obligor_key then do;
    prior_mat = 0;
    prior_any = 0;
    prev_qtr  = '';
  end;

  /* repeat flags relative to earlier quarters in the window */
  repeat_material = (material_1notch=1 and prior_mat>0);
  repeat_any      = (override_ind=1     and prior_any>0);

  /* update running state for next rows */
  if material_1notch=1 then prior_mat + 1;
  if override_ind=1     then prior_any + 1;
  prev_qtr = quarter;
run;

/* 4) Per-quarter results (this answers your question) */
proc sql;
  create table abl_material_repeat_by_qtr as
  select quarter,
         count(*)                                    as total_observations,
         sum(material_1notch)                        as mat_total,
         sum(repeat_material)                        as mat_repeat,
         calculated mat_repeat / max(calculated mat_total,1) as mat_repeat_pct format=percent8.2,
         sum(material_1notch and repeat_material=0)  as mat_first_time,
         calculated mat_first_time / max(calculated mat_total,1) as mat_first_time_pct format=percent8.2
  from _ABL_flags
  group by quarter
  order by quarter;
quit;

/* (Optional) Same idea for ANY override, not just material */
proc sql;
  create table abl_any_repeat_by_qtr as
  select quarter,
         count(*)                                as total_observations,
         sum(override_ind)                       as ovr_total,
         sum(repeat_any)                         as ovr_repeat,
         calculated ovr_repeat / max(calculated ovr_total,1) as ovr_repeat_pct format=percent8.2
  from _ABL_flags
  group by quarter
  order by quarter;
quit;

/* 5) (Optional) roster of repeated-material cases for audit */
data abl_material_repeat_roster;
  set _ABL_flags;
  if material_1notch=1 and repeat_material=1;
  keep quarter obligor_key OVRD_SEVERITY model_lgd_grade_num final_lgd_grade_num;
run;

/* 6) (Optional) Save outputs to a permanent location
libname out "/sasdata/mrmg1/MOD1497/override_results";
proc datasets lib=work nolist;
  copy in=work out=out memtype=data;
  select _ABL _ABL_sorted _ABL_flags
         abl_material_repeat_by_qtr abl_any_repeat_by_qtr
         abl_material_repeat_roster;
quit;
*/
