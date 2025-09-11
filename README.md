/* ===== Reason code stickiness using _abl_enriched + _ABL_flags ===== */
/* Assumes both tables contain: quarter, qtr_idx, c_obg, c_obgobl
   _abl_enriched : Override_Reason
   _ABL_flags    : material_1notch, repeat_material
*/

/* 1) Merge to one obligation-level table with all needed fields */
proc sort data=_abl_enriched
          out=__re1(keep=quarter qtr_idx c_obg c_obgobl Override_Reason);
  by c_obg qtr_idx c_obgobl;
run;

proc sort data=_ABL_flags
          out=__re2(keep=quarter qtr_idx c_obg c_obgobl material_1notch repeat_material);
  by c_obg qtr_idx c_obgobl;
run;

data __seq; /* obligation grain */
  merge __re1(in=a) __re2(in=b);
  by c_obg qtr_idx c_obgobl;
  if a and b;  /* keep only rows present in both */
  length reason $200;
  reason = upcase(strip(coalescec(Override_Reason,'MISSING')));
run;

/* 2) Collapse to ONE decision per obligor–quarter */
/* We compare this quarter’s MATERIAL reason(s) to the last prior MATERIAL reason
   for that obligor (carried between quarters). If any repeated material reason
   matches the prior reason, mark match_in_qtr=1. */
proc sort data=__seq; by c_obg qtr_idx c_obgobl; run;

data __obg_qtr;
  set __seq;
  by c_obg qtr_idx c_obgobl;

  retain last_mat_reason ' '  /* last prior material reason across quarters */
         has_material 0 has_repeat 0 match_in_qtr 0
         got_first_reason 0 first_mat_reason $200;

  if first.c_obg then last_mat_reason = ' ';
  if first.qtr_idx then do;
     has_material = 0; has_repeat = 0; match_in_qtr = 0;
     got_first_reason = 0; first_mat_reason = ' ';
  end;

  if material_1notch=1 then do;
     has_material = 1;
     if got_first_reason=0 then do; first_mat_reason = reason; got_first_reason=1; end;
     if repeat_material=1 then do;
        has_repeat = 1;
        if not missing(last_mat_reason) and reason = last_mat_reason then match_in_qtr = 1;
     end;
  end;

  /* emit one row per obligor-quarter and then advance the "last" reason */
  if last.qtr_idx then do;
     output;
     if has_material then last_mat_reason = first_mat_reason; /* carry forward */
  end;

  keep quarter qtr_idx c_obg has_material has_repeat match_in_qtr;
run;

/* 3) Aggregate to the final table you use: same_reason_repeat */
proc sql;
  create table same_reason_repeat as
  select quarter,
         sum(has_material)                              as mat_total,   /* # obligor-quarters with any material */
         sum(has_repeat)                                as mat_repeat,  /* # obligor-quarters where material was a repeat */
         sum(match_in_qtr)                              as same_reason, /* of those repeats, reason matched prior */
         calculated same_reason / max(calculated mat_repeat,1)
                                                        as same_reason_pct format=percent8.2
  from __obg_qtr
  group by quarter
  order by quarter;
quit;

/* (Optional) quick print */
title "Reason Code Stickiness (Same-Reason Repeats among Material Repeats)";
proc print data=same_reason_repeat noobs label; run;
title;
