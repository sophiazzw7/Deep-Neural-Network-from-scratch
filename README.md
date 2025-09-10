/* Assumes you already built WORK._ABL with: quarter (e.g., '2024Q2'),
   c_obgobl / obligor_key, override_ind, material_1notch (abs(severity)>1). */

/* 1) Make sure quarter ordering is chronological, then mark repeats */
proc sort data=_ABL out=_ABL_sorted;
  by obligor_key quarter;
run;

data _ABL_flags;
  set _ABL_sorted;
  by obligor_key quarter;
  retain prior_mat 0 prior_any 0 prev_qtr $7;

  /* reset for each obligor */
  if first.obligor_key then do;
    prior_mat = 0;
    prior_any = 0;
    prev_qtr  = '';
  end;

  /* repeat flags relative to *earlier* quarters in your window */
  repeat_material = (material_1notch=1 and prior_mat>0);
  repeat_any      = (override_ind=1     and prior_any>0);

  /* optional: consecutive-repeat flag (immediately prior quarter also material) */
  consec_material = (material_1notch=1 and prior_mat>0 and quarter ne prev_qtr and prev_qtr ne '');

  /* update running state for next rows of same obligor */
  if material_1notch=1 then prior_mat+1;
  if override_ind=1     then prior_any+1;
  prev_qtr = quarter;
run;

/* 2) Per-quarter rollups: how many of *this quarter’s* materials are repeats */
proc sql;
  create table abl_material_repeat_by_qtr as
  select quarter,
         /* dev-style denominators */
         count(*)                                    as total_observations,
         sum(material_1notch)                        as mat_total,
         sum(repeat_material)                        as mat_repeat,
         (calculated mat_repeat / max(calculated mat_total,1)) as mat_repeat_pct format=percent8.2,
         /* first-time vs repeat split */
         sum(material_1notch and not repeat_material)          as mat_first_time,
         (calculated mat_first_time / max(calculated mat_total,1)) as mat_first_time_pct format=percent8.2
  from _ABL_flags
  group by quarter
  order by quarter;
quit;

/* (Optional) If you also want the same idea for "any override", not just material: */
proc sql;
  create table abl_any_repeat_by_qtr as
  select quarter,
         count(*)                                as total_observations,
         sum(override_ind)                       as ovr_total,
         sum(repeat_any)                         as ovr_repeat,
         (calculated ovr_repeat / max(calculated ovr_total,1)) as ovr_repeat_pct format=percent8.2
  from _ABL_flags
  group by quarter
  order by quarter;
quit;
