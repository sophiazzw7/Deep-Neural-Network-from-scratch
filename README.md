/* ========= Reason-code stickiness from _flags_join2 ========= */
/* Output: same_reason_repeat  (quarter-level % of repeats with same reason) */

%macro build_reason_stick(ds=_flags_join2);
  %local dsid rc HAS_OK HAS_COBG HAS_QIDX HAS_REASON;

  /* Probe which columns actually exist */
  %let dsid=%sysfunc(open(&ds,i));
  %if &dsid %then %do;
    %let HAS_OK    = %sysfunc(varnum(&dsid,obligor_key));
    %let HAS_COBG  = %sysfunc(varnum(&dsid,c_obg));
    %let HAS_QIDX  = %sysfunc(varnum(&dsid,qtr_idx));
    %let HAS_REASON= %sysfunc(varnum(&dsid,Override_Reason));
    %let rc=%sysfunc(close(&dsid));
  %end;
  %else %do;
    %put ERROR: Cannot open &ds..;
    %return;
  %end;

  /* 1) Normalize ID / reason; ensure qtr_idx and numeric flags exist */
  data _rr_rows;
    set &ds;  /* take all columns; be defensive below */
    length obg_id $200 reason $200;

    /* choose obligor id (obligor_key -> c_obg -> c_obgobl) */
    %if &HAS_OK>0 %then %do;
      if not missing(obligor_key) then obg_id = strip(obligor_key);
      %if &HAS_COBG>0 %then %do; else if not missing(c_obg) then obg_id = strip(c_obg); %end;
      else obg_id = strip(c_obgobl);
    %end;
    %else %do;
      %if &HAS_COBG>0 %then %do; if not missing(c_obg) then obg_id = strip(c_obg); else obg_id = strip(c_obgobl); %end;
      %else %do; obg_id = strip(c_obgobl); %end;
    %end;

    /* reason text if present, else 'MISSING' */
    %if &HAS_REASON>0 %then %do;
      reason = upcase(strip(coalescec(Override_Reason,'MISSING')));
    %end;
    %else %do;
      reason = 'MISSING';
    %end;

    /* derive qtr_idx if not present (expects quarter like 2024Q3) */
    %if &HAS_QIDX=0 %then %do;
      if missing(qtr_idx) then qtr_idx = input(substr(quarter,1,4),8.)*10
                                      + input(substr(quarter,6,1),8.);
    %end;

    /* make sure flags are numeric and non-missing */
    if missing(material_1notch) then material_1notch = 0;
    if missing(repeat_material)  then repeat_material  = 0;
  run;

  /* 2) Evaluate stickiness once per obligor–quarter */
  proc sort data=_rr_rows; by obg_id qtr_idx c_obgobl; run;

  data _obg_qtr_stick;
    set _rr_rows;
    by obg_id qtr_idx c_obgobl;

    retain last_mat_reason ' ' has_mat 0 has_rpt 0 match_in_qtr 0
           got_first 0 first_mat_reason $200;

    if first.obg_id then last_mat_reason = ' ';
    if first.qtr_idx then do;
      has_mat=0; has_rpt=0; match_in_qtr=0;
      got_first=0; first_mat_reason=' ';
    end;

    /* Only for MATERIAL rows */
    if material_1notch=1 then do;
      has_mat = 1;
      if got_first=0 then do; first_mat_reason = reason; got_first=1; end;
      if repeat_material=1 then do;
        has_rpt = 1;
        if not missing(last_mat_reason) and reason = last_mat_reason then match_in_qtr = 1;
      end;
    end;

    /* one row per obligor–quarter; carry forward this quarter's reason */
    if last.qtr_idx then do;
      output;
      if has_mat then last_mat_reason = first_mat_reason;
    end;

    keep quarter qtr_idx obg_id has_mat has_rpt match_in_qtr;
  run;

  /* 3) Final quarter-level table */
  proc sql;
    create table same_reason_repeat as
    select quarter,
           sum(has_mat)      as mat_total,    /* # obligor-quarters with material */
           sum(has_rpt)      as mat_repeat,   /* # obligor-quarters where material was repeat */
           sum(match_in_qtr) as same_reason,  /* repeats whose reason matched prior */
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
%build_reason_stick(ds=_flags_join2);
