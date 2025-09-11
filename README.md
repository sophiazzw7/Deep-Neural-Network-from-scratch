/* ---- FIXED join: use c_obg for repeat logic; c_obgobl for exposure ---- */
proc sql;
  create table _flags_join2 as
  select
      f.quarter,
      f.c_obg,                 /* obligor key for repeat logic */
      f.c_obgobl,              /* obligor+obligation key for exposure */
      f.qtr_idx,
      f.material_1notch,
      f.repeat_material,
      f.OVRD_SEVERITY,
      a.tfc_face_amt,
      a.TFC_Curr_Bal,
      a.legacy_bank_new,
      a.STI_lob_nm,
      a.STI_sub_lob_nm
  from _abl_flags f
  left join _abl_attrs a
    on  f.quarter  = a.quarter
    and f.c_obgobl = a.c_obgobl
  ;
quit;
2) Direction (use material only for up/down splits)
sas
Copy code
/* ---- Direction, consistently filtered to material (abs severity > 1) ---- */
proc sql;
  create table direction_consistent as
  select
      quarter,
      sum(material_1notch=1)                                        as mat_total,
      sum(material_1notch=1 and OVRD_SEVERITY>0)                    as mat_downgrades,
      sum(material_1notch=1 and OVRD_SEVERITY<0)                    as mat_upgrades
  from _flags_join2
  group by quarter
  order by quarter;
quit;

/* optional print (keeps your title style) */
title "Direction (consistent with material totals)";
proc print data=direction_consistent noobs;
  var quarter mat_downgrades mat_upgrades mat_total;
run; title;
3) Exposure share (sum exposure on material rows; denom = all rows)
sas
Copy code
/* ---- Exposure share computed at obligation level via c_obgobl join ---- */
proc sql;
  create table abl_exposure_share as
  select
      quarter,
      sum(tfc_face_amt)                                              as face_total,
      sum(case when material_1notch=1 then tfc_face_amt  else 0 end) as face_material,
      calculated face_material / max(calculated face_total,1)        as face_mat_share format=percent8.2,
      sum(TFC_Curr_Bal)                                              as bal_total,
      sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end)  as bal_material,
      calculated bal_material / max(calculated bal_total,1)          as bal_mat_share  format=percent8.2
  from _flags_join2
  group by quarter
  order by quarter;
quit;

/* optional print */
title "Exposure Share of Material";
proc print data=abl_exposure_share noobs;
  var quarter face_total face_material face_mat_share bal_total bal_material bal_mat_share;
run; title;
4) Reasons (top-10 per quarter) + optional “repeats-only” view
sas
Copy code
/* ---- All material reasons, by quarter ---- */
proc sql;
  create table abl_by_reason as
  select
      quarter,
      coalesce(Override_Reason,'(missing)')                 as reason length=200,
      count(*)                                              as obs,
      sum(material_1notch)                                  as mat_cnt,
      calculated mat_cnt / max(calculated obs,1)            as mat_rate format=percent8.2
  from _flags_join2
  group by quarter, reason
  order by quarter, mat_cnt desc;
quit;

/* print top 10 per quarter (same as your chart) */
title "Material Override Reasons (top 10 by material count)";
proc sort data=abl_by_reason out=_t3;
  by quarter descending mat_cnt;
run;
proc print data=_t3(obs=10) noobs;
  var quarter reason obs mat_cnt mat_rate;
run; title;

/* ---- OPTIONAL: reason mix among repeats only ---- */
proc sql;
  create table abl_by_reason_repeats as
  select
      quarter,
      coalesce(Override_Reason,'(missing)')                 as reason length=200,
      count(*)                                              as obs,
      sum(material_1notch and repeat_material)              as rpt_mat_cnt,
      calculated rpt_mat_cnt / max(calculated obs,1)        as rpt_mat_rate format=percent8.2
  from _flags_join2
  group by quarter, reason
  order by quarter, rpt_mat_cnt desc;
quit;
/* print top 10 among repeats */
title "Material Reasons among Repeat Overrides (top 10)";
proc sort data=abl_by_reason_repeats out=_t3r;
  by quarter descending rpt_mat_cnt;
run;
proc print data=_t3r(obs=10) noobs;
  var quarter reason obs rpt_mat_cnt rpt_mat_rate;
run; title;
5) Repeat overrides by quarter (ties to your current flags; kept intact)
sas
Copy code
/* ---- Repeat material vs first-time material by quarter ---- */
proc sql;
  create table abl_material_repeat_by_qtr as
  select
      quarter,
      count(*)                                        as total_observations,
      sum(material_1notch=1)                          as mat_total,
      sum(material_1notch=1 and repeat_material=1)    as mat_repeat,
      calculated mat_repeat / max(calculated mat_total,1) as mat_repeat_pct format=percent8.2,
      sum(material_1notch=1 and repeat_material=0)    as mat_first_time,
      calculated mat_first_time / max(calculated mat_total,1) as mat_first_time_pct format=percent8.2
  from _flags_join2
  group by quarter
  order by quarter;
quit;

/* optional print */
title "Material Overrides Repeat by Quarter";
proc print data=abl_material_repeat_by_qtr noobs label;
  var quarter total_observations mat_total mat_repeat mat_repeat_pct mat_first_time mat_first_time_pct;
run; title;
