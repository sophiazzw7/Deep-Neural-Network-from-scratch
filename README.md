/* ===== Reason-code stickiness from _flags_join2 (defensive, no qtr_idx needed) ===== */
/* Requires in _flags_join2: QUARTER, C_OBGOBL, MATERIAL_1NOTCH, REPEAT_MATERIAL
   Optional: OBLIGOR_KEY and/or C_OBG, OVERRIDE_REASON                                  */

%macro reason_stick_from_flags(ds=_flags_join2);

  /* Discover which ID/Reason columns exist */
  %local dsid rc has_ok has_cobg has_reason has_quarter has_mat has_rpt has_obgobl;
  %let dsid=%sysfunc(open(&ds));
  %if &dsid = 0 %then %do; %put ERROR: Cannot open &ds..; %return; %end;

  %let has_ok      = %sysfunc(varnum(&dsid,obligor_key));
  %let has_cobg    = %sysfunc(varnum(&dsid,c_obg));
  %let has_reason  = %sysfunc(varnum(&dsid,Override_Reason));
  %let has_quarter = %sysfunc(varnum(&dsid,quarter));
  %let has_mat     = %sysfunc(varnum(&dsid,material_1notch));
  %let has_rpt     = %sysfunc(varnum(&dsid,repeat_material));
  %let has_obgobl  = %sysfunc(varnum(&dsid,c_obgobl));
  %let rc=%sysfunc(close(&dsid));

  /* Minimal checks */
  %if &has_quarter=0 or &has_mat=0 or &has_rpt=0 or &has_obgobl=0 %then %do;
    %put ERROR: &ds must contain QUARTER, C_OBGOBL, MATERIAL_1NOTCH, REPEAT_MATERIAL.;
    %return;
  %end;

  /* Choose an obligor identifier */
  %if &has_ok>0 %then %let IDVAR=obligor_key;
  %else %if &has_cobg>0 %then %let IDVAR=c_obg;
  %else %let IDVAR=c_obgobl;           /* fallback (obligation-level) */

  /* 1) Normalize ID & REASON (treat missing reason as 'MISSING') */
  data _rr_rows;
    set &ds;
    length obg_id $200 reason $200;
    obg_id = strip(&IDVAR);
    %if &has_reason>0 %then %do;
      reason = upcase(strip(coalescec(Override_Reason,'MISSING')));
    %end;
    %else %do;
      reason = 'MISSING';
    %end;

    /* ensure flags are numeric */
    if missing(material_1notch) then material_1notch = 0;
    if missing(repeat_material)  then repeat_material  = 0;
  run;

  /* 2) Evaluate stickiness once per obligor–quarter (order by QUARTER string) */
  proc sort data=_rr_rows; by obg_id quarter c_obgobl; run;

  data _obg_qtr_stick;
    set _rr_rows;
    by obg_id quarter c_obgobl;

    retain last_mat_reason ' ' has_mat 0 has_rpt 0 match_in_qtr 0
           got_first 0 first_mat_reason $200;

    if first.obg_id then last_mat_reason = ' ';
    if first.quarter then do;
      has_mat=0; has_rpt=0; match_in_qtr=0;
      got_first=0; first_mat_reason=' ';
    end;

    /* consider only MATERIAL rows */
    if material_1notch=1 then do;
      has_mat = 1;

      /* capture first material reason this quarter (to carry forward) */
      if got_first=0 then do; first_mat_reason = reason; got_first=1; end;

      if repeat_material=1 then do;
        has_rpt = 1;
        /* treat missing reason as non-match to avoid inflating stickiness */
        if not missing(last_mat_reason) and last_mat_reason^='MISSING'
           and reason^='MISSING' and reason = last_mat_reason
        then match_in_qtr = 1;
      end;
    end;

    /* emit one row per obligor–quarter; then advance last_mat_reason */
    if last.quarter then do;
      output;
      if has_mat then last_mat_reason = first_mat_reason;
    end;

    keep quarter obg_id has_mat has_rpt match_in_qtr;
  run;

  /* 3) Quarter-level table you want */
  proc sql;
    create table same_reason_repeat as
    select quarter,
           sum(has_mat)      as mat_total,
           sum(has_rpt)      as mat_repeat,
           sum(match_in_qtr) as same_reason,
           calculated same_reason / max(calculated mat_repeat,1)
             as same_reason_pct format=percent8.2
    from _obg_qtr_stick
    group by quarter
    order by quarter;
  quit;

  title "Reason Code Stickiness (Same-Reason Repeats among Material Repeats)";
  proc print data=same_reason_repeat noobs label; run;
  title;

%mend;

/* Run it */
%reason_stick_from_flags(ds=_flags_join2);
