/* ===== Obligation + LGD approval repeat (minimal add-on) ===== */
/* Assumes _ABL already has: quarter, qtr_idx, material_1notch, OVRD_SEVERITY,
   and (from step 1) STI_obl_nbr, c_obl, fpb_lgdappid              */

data _ABL_oblapp_prep;
  length obligation_key $40 lgd_app_id $40;
  set _ABL;

  /* Build obligation_key (prefer STI_obl_nbr, else c_obl) as character */
  if not missing(STI_obl_nbr) then obligation_key = strip(put(STI_obl_nbr,best16.));
  else if not missing(c_obl)   then obligation_key = strip(put(c_obl,best16.));
  else obligation_key='';

  /* LGD approval id as character */
  if not missing(fpb_lgdappid) then lgd_app_id = strip(put(fpb_lgdappid,best16.));
  else lgd_app_id='';

  /* Drop rows without a composite key */
  if missing(obligation_key) or missing(lgd_app_id) then delete;
run;

proc sort data=_ABL_oblapp_prep out=_ABL_oblapp_sorted;
  by obligation_key lgd_app_id qtr_idx;
run;

/* Flag repeats for the SAME (obligation, LGD approval) across any prior quarter */
data ABL_flags_oblapp;
  set _ABL_oblapp_sorted;
  by obligation_key lgd_app_id;

  retain prior_mat 0;
  if first.lgd_app_id then prior_mat=0;

  repeat_material_oblapp = 0;
  if material_1notch=1 then do;
    if prior_mat>0 then repeat_material_oblapp=1;
    prior_mat+1;
  end;
run;

/* Quarter summary at obligation+LGD level (names parallel your existing table) */
proc sql;
  create table abl_material_repeat_by_qtr_oblapp as
  select quarter,
         count(*)                                        as total_observations,
         sum(material_1notch)                            as mat_total,
         sum(repeat_material_oblapp)                     as mat_repeat,
         sum(material_1notch)-sum(repeat_material_oblapp) as mat_first_time,
         calculated mat_repeat     / max(calculated mat_total,1) as mat_repeat_pct     format=percent8.2,
         calculated mat_first_time / max(calculated mat_total,1) as mat_first_time_pct format=percent8.2
  from ABL_flags_oblapp
  group by quarter
  order by quarter;
quit;

/* Optional quick print to mirror your current output style */
title "Material Overrides Repeat by Quarter (Obligation + LGD Approval)";
proc print data=abl_material_repeat_by_qtr_oblapp noobs label;
  var quarter total_observations mat_total mat_repeat mat_repeat_pct
      mat_first_time mat_first_time_pct;
run;
title;
