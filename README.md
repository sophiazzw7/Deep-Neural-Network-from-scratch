/* ====== MOD1497 repeat override analysis using developer DEDUP tables ====== */

options compress=yes reuse=yes mprint;

/* EDIT: where you saved the DEDUP datasets */
libname ogm  "/sasdata/mrmg1/MOD1497/override_dedup";   /* <- your folder */
libname out  "/sasdata/mrmg1/MOD1497/override_results"; /* outputs */

/* Quarters you saved (labels must match the dataset names) */
%let qtrs = 2024Q2 2024Q3 2024Q4 2025Q1;
%let segs = ABL SF;

/* Stack all quarters and segments */
%macro stack_all;
  %do s=1 %to %sysfunc(countw(&segs));
    %let seg=%scan(&segs,&s);

    data work._&seg;
      length quarter $7 seg $3 obligor_key $64;
      set
      %do i=1 %to %sysfunc(countw(&qtrs));
        %let q=%scan(&qtrs,&i);
        %if %sysfunc(exist(ogm.OVERRIDE_&q._DEDUP_&seg)) %then %do;
          ogm.OVERRIDE_&q._DEDUP_&seg (in=in&i
            keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                 model_lgd_grade_num final_lgd_grade_num Override_Reason)
        %end;
      %end;
      ;
      seg="&seg";
      obligor_key = c_obgobl;
      %do i=1 %to %sysfunc(countw(&qtrs));
        %let q=%scan(&qtrs,&i);
        if in&i then quarter="&q";
      %end;

      /* If OVRD_SEVERITY was not carried, re-compute from numeric grades */
      if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0 then
        OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;

      /* Developer material definition */
      material_1notch = (abs(OVRD_SEVERITY) > 1);

      /* Safety: if override_ind missing but severity non-zero, set it */
      if missing(override_ind) and not missing(OVRD_SEVERITY) then
        override_ind = (OVRD_SEVERITY ne 0);
    run;
  %end;

  data work.allq;
    set work._ABL work._SF;
  run;
%mend;
%stack_all;

/* -------- Summary by quarter × segment -------- */
proc sql;
  create table out.ovr_summary_by_qtr_seg as
  select quarter, seg,
         count(*)                                    as obs_cnt,
         sum(override_ind)                            as ovr_cnt,
         calculated ovr_cnt / calculated obs_cnt      as ovr_rate format=percent8.2,
         sum(material_1notch)                         as mat_cnt,
         calculated mat_cnt / calculated obs_cnt      as mat_rate format=percent8.2
  from work.allq
  group by quarter, seg
  order by quarter, seg;
quit;

/* -------- Repeats across quarters (obligor-level) -------- */
proc sql;
  create table work._per_obligor as
  select seg, obligor_key,
         count(distinct quarter)                    as qtrs_seen,
         sum(override_ind>0)                        as qtrs_overridden,
         sum(material_1notch>0)                     as qtrs_material,
         min(quarter)                               as first_qtr,
         max(quarter)                               as last_qtr
  from work.allq
  group by seg, obligor_key;
quit;

data out.repeat_overrides_any;
  set work._per_obligor;
  where qtrs_overridden >= 2;  /* repeated overrides */
run;

data out.repeat_overrides_material;
  set work._per_obligor;
  where qtrs_material >= 2;    /* repeated MATERIAL overrides */
run;

/* -------- Optional detail table for audit trail -------- */
proc sort data=work.allq out=out.obligor_quarter_detail;
  by seg obligor_key quarter;
run;

/* Quick prints */
title "MOD1497 Override & Material Rates by Quarter × Segment";
proc print data=out.ovr_summary_by_qtr_seg noobs; run;

title "Obligors with Repeated Overrides (≥2 quarters)";
proc print data=out.repeat_overrides_any(obs=100) noobs; run;

title "Obligors with Repeated MATERIAL Overrides (≥2 quarters)";
proc print data=out.repeat_overrides_material(obs=100) noobs; run;
title;
/* Define a permanent library where you want to store outputs */
libname ogm "/sasdata/mrmg1/MOD1497/override_dedup";  

/* Copy dedup tables from WORK into your permanent library */
proc datasets lib=work nolist;
  copy in=work out=ogm memtype=data;
  select 
    OVERRIDE_2024Q2_DEDUP_ABL
    OVERRIDE_2024Q2_DEDUP_SF
    OVERRIDE_2024Q3_DEDUP_ABL
    OVERRIDE_2024Q3_DEDUP_SF
    OVERRIDE_2024Q4_DEDUP_ABL
    OVERRIDE_2024Q4_DEDUP_SF
    OVERRIDE_2025Q1_DEDUP_ABL
    OVERRIDE_2025Q1_DEDUP_SF;
quit;
