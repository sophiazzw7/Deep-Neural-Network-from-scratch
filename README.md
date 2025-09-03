/*==================== USER INPUTS (edit paths/quarters if needed) ====================*/
%let ROOT = /sasdata/mrmg1/MOD1497;
%let QLIST = 2024Q2_v2.1 2024Q3_v2.1 2024Q4_v2.1 2025Q1_v2.2;

/* Dataset name inside each folder (rename here if different) */
%let JOINDS = join_1497_final_res;   /* expects variables:
                                        - model_lgd_grade, final_lgd_grade
                                        - Override_Reason (character codes)
                                        - tfc_face_amt (commitment)
                                        - TFC_Account_Nbr or TFC_Account_Nbr_New (facility id)
                                        - SF (0/1) or a way to derive segment
                                        - STI_lob_nm (optional, for fallback)  */

/* List of "operational" override reasons to EXCLUDE from override counting (from OGM code) */
%let OR_OPS = 3 4 5 6 7 32 33 36 37 38;

/*==================== 1) MAP GRADES TO NUMERIC FOR SEVERITY ====================*/
proc format;
  value $lgd_num
    'A'=1  'B'=2  'C'=3  'D'=4  'E'=5  'F'=6  'G'=7  'H'=8  'I'=9  'J'=10
    'K'=11 'L'=12 'M'=13 'N'=14 'O'=15 'P'=16 'Q'=17 'R'=18 'S'=19 'T'=20;
run;

/*==================== 2) STACK QUARTERS ====================*/
data OV_DISC_RAW;
  length OGM_Qtr $12 source_path $200;
  format OGM_Qtr $12.;
  %local i q path;
  %let i=1;
  %do %while(%scan(&QLIST,&i,%str( )) ne );
    %let q    = %scan(&QLIST,&i,%str( ));
    %let path = &ROOT./&q.;
    libname q&i "&path";

    /* If the dataset lives in a subfolder, update the libref or add a second libname here */

    /* Bring in the quarter and keep only rows with positive commitment */
    data _q&i;
      length OGM_Qtr $12 source_path $200;
      set q&i..&JOINDS (in=ok);
      if not ok then stop;

      OGM_Qtr = "&q";
      source_path = "&path";

      /* -------- facility id (edit if your ID differs) -------- */
      length fac_id $40;
      fac_id = coalescec(strip(TFC_Account_Nbr_New), strip(TFC_Account_Nbr));

      /* -------- segment: prefer SF flag; fallback using STI_lob_nm if needed -------- */
      length segment $3;
      if not missing(SF) then segment = ifc(SF=1,'SF','ABL');
      else if upcase(STI_lob_nm) in ('CONSUMER CREDIT','RETAIL BANKING') then segment='SF';
      else segment='ABL';

      /* -------- override severity & indicators -------- */
      length model_lgd_grade final_lgd_grade $2;
      model_num = input(put(strip(model_lgd_grade),$lgd_num.), best12.);
      final_num = input(put(strip(final_lgd_grade),$lgd_num.), best12.);
      if missing(model_num) or missing(final_num) then sev = .;
      else sev = final_num - model_num;  /* + = downgrade, - = upgrade */

      /* Exclude purely operational overrides from counting */
      length Override_Reason $20;
      retain override_ops 0;
      override_ops = 0;
      /* Normalize reason to numeric if stored as char digits */
      _or_num = input(strip(Override_Reason), best12.);
      if _or_num in (&OR_OPS) then override_ops=1;

      any_override     = (final_lgd_grade ne model_lgd_grade) and (override_ops=0);
      onotch_upgrade   = (any_override=1 and sev <= -1 and sev > -2);
      onotch_downgrade = (any_override=1 and sev >=  1 and sev <  2);
      material_ovr     = (any_override=1 and abs(sev) >= 2);

      /* commitment & drawn (drawn not needed for override rate, but may help explain) */
      commit = tfc_face_amt;
      drawn  = tfc_curr_bal;  /* if present; ok if missing */

      if commit>0;  /* mirror OGM filter */
      drop _or_num;
    run;

    %if &i=1 %then %do;
      data OV_DISC_RAW; set _q&i; run;
    %end;
    %else %do;
      proc append base=OV_DISC_RAW data=_q&i force; run;
    %end;

    %let i=%eval(&i+1);
  %end;
run;

/*==================== 3) QUARTER x SEGMENT RATES (Any / Material / 1-notch) ====================*/
proc sql;
  create table OV_RATE as
  select OGM_Qtr,
         segment,
         count(distinct fac_id)                             as n_facilities,
         sum(any_override)                                  as n_any,
         sum(material_ovr)                                  as n_material,
         sum(onotch_upgrade + onotch_downgrade)             as n_onotch,
         calculated n_any      / calculated n_facilities    as pct_any      format=percent8.2,
         calculated n_material / calculated n_facilities    as pct_material format=percent8.2,
         calculated n_onotch   / calculated n_facilities    as pct_onotch   format=percent8.2
  from OV_DISC_RAW
  group by 1,2
  order by OGM_Qtr, segment;
quit;

/*==================== 4) REPEAT-OVERRIDE CHECK (same facility across quarters) ====================*/
proc sql;
  /* facilities that have an override in >=2 quarters */
  create table REPEAT_FAC as
  select fac_id, segment,
         count(*) as n_quarters_with_ovr,
         min(OGM_Qtr) as first_qtr,
         max(OGM_Qtr) as last_qtr
  from OV_DISC_RAW
  where any_override=1
  group by fac_id, segment
  having calculated n_quarters_with_ovr >= 2
  order by n_quarters_with_ovr desc, fac_id;

  /* share of overrides coming from repeat facilities by quarter/segment */
  create table REPEAT_SHARE as
  select a.OGM_Qtr, a.segment,
         count(distinct a.fac_id) as n_fac_ovr,
         sum(case when r.fac_id is not null then 1 else 0 end) as n_fac_repeat,
         calculated n_fac_repeat / calculated n_fac_ovr as pct_repeat_fac format=percent8.2
  from (select distinct OGM_Qtr, segment, fac_id
        from OV_DISC_RAW where any_override=1) as a
  left join (select distinct fac_id, segment from REPEAT_FAC) as r
    on a.fac_id=r.fac_id and a.segment=r.segment
  group by 1,2
  order by 1,2;
quit;

/*==================== 5) OPTIONAL DETAIL: SEVERITY DISTRIBUTION ====================*/
proc sql;
  create table OV_SEV_DIST as
  select OGM_Qtr, segment,
         sum(any_override)                                  as n_any,
         sum(case when abs(sev)<1 then 0 else 1 end)        as n_ge1,
         sum(case when abs(sev)>=2 then 1 else 0 end)       as n_ge2,
         calculated n_ge1 / calculated n_any                as pct_ge1 format=percent8.2,
         calculated n_ge2 / calculated n_any                as pct_ge2 format=percent8.2
  from OV_DISC_RAW
  group by 1,2
  order by OGM_Qtr, segment;
quit;

/*==================== 6) QUICK PRINTS ====================*/
title "Override Rates by Quarter/Segment";
proc print data=OV_RATE noobs; run;

title "Repeat Facilities (>=2 quarters with override)";
proc print data=REPEAT_FAC(obs=50) noobs; run;

title "Share of Overrides from Repeat Facilities";
proc print data=REPEAT_SHARE noobs; run;

title "Override Severity Distribution";
proc print data=OV_SEV_DIST noobs; run;
title;
