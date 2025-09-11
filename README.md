/* ===== Reason-code stickiness using _flags_join2 =====
   _flags_join2 should contain at least:
     quarter, qtr_idx, c_obgobl,
     material_1notch, repeat_material,
     Override_Reason,
     and an obligor identifier (obligor_key preferred, else c_obg; we’ll fall back to c_obgobl if needed)
*/

/* 1) Normalize ID and reason text */
data _rr_rows;
  set _flags_join2(keep=quarter qtr_idx c_obgobl material_1notch repeat_material
                          Override_Reason obligor_key c_obg);
  length obg_id $200 reason $200;
  /* choose an obligor id */
  if not missing(obligor_key) then obg_id = strip(obligor_key);
  else if not missing(c_obg)   then obg_id = strip(c_obg);
  else                              obg_id = strip(c_obgobl);   /* last resort */
  reason = upcase(strip(coalescec(Override_Reason,'MISSING')));
run;

/* 2) One decision per obligor–quarter:
      Does any MATERIAL REPEAT this quarter use the SAME reason as the last prior MATERIAL reason?
*/
proc sort data=_rr_rows; by obg_id qtr_idx c_obgobl; run;

data _obg_qtr_stick;
  set _rr_rows;
  by obg_id qtr_idx c_obgobl;

  retain last_mat_reason ' '              /* last prior MATERIAL reason for this obligor */
         has_mat 0 has_rpt 0 match_in_qtr 0
         got_first 0 first_mat_reason $200;

  if first.obg_id then last_mat_reason = ' ';
  if first.qtr_idx then do;
    has_mat = 0; has_rpt = 0; match_in_qtr = 0;
    got_first = 0; first_mat_reason = ' ';
  end;

  if material_1notch = 1 then do;         /* only evaluate MATERIAL rows */
    has_mat = 1;
    if got_first = 0 then do; first_mat_reason = reason; got_first = 1; end;

    if repeat_material = 1 then do;
      has_rpt = 1;
      if not missing(last_mat_reason) and reason = last_mat_reason then match_in_qtr = 1;
    end;
  end;

  /* emit one row per obligor–quarter; carry forward this quarter's first material reason */
  if last.qtr_idx then do;
    output;
    if has_mat then last_mat_reason = first_mat_reason;
  end;

  keep quarter qtr_idx obg_id has_mat has_rpt match_in_qtr;
run;

/* 3) Quarter-level stickiness table (Same-Reason Repeats among Material Repeats) */
proc sql;
  create table same_reason_repeat as
  select quarter,
         sum(has_mat)          as mat_total,    /* obligor-quarters with any MATERIAL */
         sum(has_rpt)          as mat_repeat,   /* obligor-quarters where MATERIAL was a REPEAT */
         sum(match_in_qtr)     as same_reason,  /* of those repeats, reason matched prior */
         calculated same_reason / max(calculated mat_repeat,1)
                               as same_reason_pct format=percent8.2
  from _obg_qtr_stick
  group by quarter
  order by quarter;
quit;

/* (Optional) quick view */
title "Reason Code Stickiness (Same-Reason Repeats among Material Repeats)";
proc print data=same_reason_repeat noobs label; run;
title;
