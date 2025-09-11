/* ===== Check repeats for the existing Top-10 reason codes ===== */
/* 1) Recompute reason stickiness safely from abl_enriched */
proc sort data=abl_enriched out=_rb_prep;
  by c_obg qtr_idx c_obgobl;
run;

data _reason_base;
  set _rb_prep(keep=quarter qtr_idx c_obg c_obgobl
                     material_1notch repeat_material Override_Reason);
  by c_obg qtr_idx c_obgobl;
  length reason $200 same_reason_repeat 8;
  retain prev_reason;

  if first.c_obg then prev_reason='';

  /* normalize reason */
  reason = coalescec(strip(Override_Reason),'MISSING');

  /* default numeric 0 so SUM() works */
  same_reason_repeat = 0;

  /* only evaluate stickiness for material repeats */
  if material_1notch=1 then do;
    if repeat_material=1 and not missing(prev_reason) and
       upcase(strip(reason)) = upcase(strip(prev_reason)) then same_reason_repeat=1;
    /* update last material reason for this obligor */
    prev_reason = reason;
  end;
run;

/* 2) Aggregate repeat metrics by quarter/reason */
proc sql;
  create table _reason_metrics as
  select quarter,
         reason,
         count(*)                  as obs,
         sum(material_1notch)      as mat_cnt,
         sum(repeat_material)      as mat_repeat,
         sum(same_reason_repeat)   as same_reason_repeat,
         calculated same_reason_repeat / max(calculated mat_repeat,1)
                                     as same_reason_pct format=percent8.2
  from _reason_base
  group by quarter, reason;
quit;

/* 3) Keep only the quarter/reason in your existing Top-10 table */
proc sql;
  create table top10_reason_repeats as
  select t.quarter,
         t.reason,
         m.obs,
         m.mat_cnt,
         m.mat_repeat,
         m.same_reason_repeat,
         m.same_reason_pct
  from abl_reason_top10 as t
  left join _reason_metrics as m
    on t.quarter = m.quarter
   and upcase(strip(t.reason)) = upcase(strip(m.reason))
  order by t.quarter, t.rank_within_qtr;
quit;

/* 4) Print */
title "Repeat Check for Top-10 Material Override Reasons (per Quarter)";
proc print data=top10_reason_repeats noobs label;
  var quarter reason obs mat_cnt mat_repeat same_reason_repeat same_reason_pct;
run;
title;
