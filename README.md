/* ============================================================= */
/*            O V E R R I D E   A N A L Y S I S  (OGM)           */
/*        Mirrors developer OGM logic/skeleton exactly            */
/*  – last-12-months window; SF uses imp_dtsf lower bound         */
/*  – exclusions: model/final empty, operational reasons          */
/*  – de-dup by c_obgobl + f_uplddt (keep LAST)                   */
/*  – “1-NOTCH” materiality = ABS(OVRD_SEVERITY) > 1              */
/*  – heritage & total rollups, same computed fields              */
/*  Only aliasing done: Override_Reason, c_obgobl, tfc_face_amt   */
/* ============================================================= */

options compress=yes reuse=yes;

/* -------- Repo locations (your MOD1497 folders) -------- */
%let ROOT=/sasdata/mrmg1/MOD1497;
libname q2  "&ROOT/2024Q2_v2.1";
libname q3  "&ROOT/2024Q3_v2.1";
libname q4  "&ROOT/2024Q4_v2.1";
libname q1  "&ROOT/2025Q1_v2.2";

/* For exact OGM dataset naming below */
libname New_OGM work;

/* =========================================
   Canonical stack to match OGM “all_table”
   ========================================= */
data work__stack_raw;
  length OGM_Qtr $6 Seg $2;
  set
    q2.join_mod1497_tot_2024q2_abl (in=a1) 
    q2.join_mod1497_tot_2024q2_sf  (in=a2)
    q3.join_mod1497_tot_2024q3_abl (in=b1)
    q3.join_mod1497_tot_2024q3_sf  (in=b2)
    q4.join_mod1497_tot_2024q4_abl (in=c1)
    q4.join_mod1497_tot_2024q4_sf  (in=c2)
    q1.join_mod1497_tot_2025q1_abl (in=d1)
    q1.join_mod1497_tot_2025q1_sf  (in=d2)
  ;

  if a1 or a2 then OGM_Qtr='2024Q2';
  else if b1 or b2 then OGM_Qtr='2024Q3';
  else if c1 or c2 then OGM_Qtr='2024Q4';
  else if d1 or d2 then OGM_Qtr='2025Q1';

  Seg = ifc(a2 or b2 or c2 or d2,'sf','abl');
run;

/* === Alias layer (only to line up OGM field names) ============== */
/* - Override_Reason: your tables use model_or_reason               */
/* - c_obgobl: use existing char key if present, else cats(c_obg,_) */
/* - tfc_face_amt: if absent, use face_amt_mtd1 (seen in screenshots) */
data New_OGM.all_table_6ymm_1;
  set work__stack_raw;

  length Override_Reason $10 c_obgobl $200;
  if missing(Override_Reason) then Override_Reason = strip(model_or_reason);

  if not missing(c_obgobl) then c_obgobl = strip(c_obgobl);
  else if not missing(c_obg_obl) then c_obgobl = strip(c_obg_obl);
  else c_obgobl = cats(c_obg,'_',c_obl);

  if nmiss(tfc_face_amt)=1 and not missing(face_amt_mtd1) then tfc_face_amt = face_amt_mtd1;
run;

/* --------------- LGD letter->number (A=1 … T=20) ---------------- */
proc format;
  value $lgd_num
    'A'=1  'B'=2  'C'=3  'D'=4  'E'=5  'F'=6  'G'=7  'H'=8  'I'=9  'J'=10
    'K'=11 'L'=12 'M'=13 'N'=14 'O'=15 'P'=16 'Q'=17 'R'=18 'S'=19 'T'=20
  ;
run;

/* ================================================================= */
/* Macro OR mirrors your OGM code exactly (from your screenshots)    */
/* ================================================================= */
%macro OR(seg=abl, OGM_Qtr=2025Q1, OGM_Cutoff='31MAR2025'd, ymm=6ymm, imp_dtsf='01JAN2024'd);

  /* last 12 months per MDD */
  %let OR_window_stat=%sysfunc(intnx(month,&OGM_Cutoff,-11,BEG),date9.);
  %put NOTE: OR_window_stat = &OR_window_stat;

  /* ---------------- Prepare rated obligations ---------------- */
  data Override_&OGM_Qtr._&Seg;
    set New_OGM.all_table_&ymm._1;
    %if &seg.=sf %then %do;
      where model_lgd_grade^='' and final_lgd_grade^='' 
            and &imp_dtsf.<=f_uplddt<=&OGM_Cutoff.
            and tfc_face_amt>0;
    %end;
    %else %do;
      where model_lgd_grade^='' and final_lgd_grade^='' 
            and "&OR_window_stat."d < f_uplddt <= &OGM_Cutoff.
            and tfc_face_amt>0;
    %end;

    /* match OGM: remove Retail/Consumer LOB; keep only model A–R */
    if STI_lob_NM not in ('Consumer Credit','Retail Banking') 
       and model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');

    /* segment flag like OGM */
    %if &seg.=sf %then %do; SF=1; %end;
    %else %do; SF=0; %end;
  run;

  /* Exclusion 5) remove operational reasons (exact list from your code) */
  data Override_&OGM_Qtr._&Seg;
    set Override_&OGM_Qtr._&Seg;
    /* numericize grades and compute severity */
    model_lgd_grade_num = 1*put(model_lgd_grade,$lgd_num.);
    final_lgd_grade_num = 1*put(final_lgd_grade,$lgd_num.);
    if final_lgd_grade^=model_lgd_grade then override_ind=1; else override_ind=0;
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;  /* + = downgrade, - = upgrade */
  run;

  /* keep only non-operational override rows for metrics in their code path */
  data Override_&OGM_Qtr._&Seg;
    set Override_&OGM_Qtr._&Seg;
    if Override_Reason not in ('3','4','5','6','7','32','33','36','37','38'); /* Operational Override Reasons */
  run;

  /* Quick reason and window sanity (as in their code) */
  proc freq data=New_OGM.all_table_&ymm._1; tables Override_Reason; run;
  proc freq data=Override_&OGM_Qtr._&Seg; tables f_uplddt*Override_Reason / norow nocol; run;

  /* --------- De-dup: by c_obgobl then f_uplddt, keep LAST --------- */
  proc sort data=Override_&OGM_Qtr._&Seg;
    by c_obgobl f_uplddt;
  run;

  data Override_&OGM_Qtr._dedup_&Seg;
    set Override_&OGM_Qtr._&Seg;
    by c_obgobl f_uplddt;
    if last.c_obgobl then output;
  run;

  /* Checking dup counts like your SQL block */
  proc sql;
    select c_obgobl, count(*) as CNT
    from Override_&OGM_Qtr._dedup_&Seg
    group by c_obgobl
    having count(*) > 1
  ;quit;

  /* If running “all”, concatenate abl+sf */
  %if &seg.=all %then %do;
    data Override_&OGM_Qtr._dedup_all;
      set Override_&OGM_Qtr._dedup_abl Override_&OGM_Qtr._dedup_sf;
    run;
  %end;

  /* =================== Table 1: Override Analysis =================== */
  proc sql;
    create table Override_Distribution_&Seg as
    select
      legacy_bank_new,
      model_lgd_grade,
      final_lgd_grade,
      count(1) as cnt
    from Override_&OGM_Qtr._dedup_&Seg
    group by 1,2,3
    ;
  quit;

  /*************** Heritage-level Override Reporting ******************/
  proc sql;
    create table OVERRIDE_Sum_heritage_&Seg as
    select
      legacy_bank_new,
      count(*)                                   as TOTAL_CNT format=comma15.0,
      sum(override_ind)                          as OVRD_CNT  format=comma15.0,
      sum(case when OVRD_SEVERITY >  0 then override_ind else 0 end) as Downgrade,
      sum(case when OVRD_SEVERITY <  0 then override_ind else 0 end) as Upgrade,
      calculated Downgrade / calculated TOTAL_CNT as Downgrade_PCT format=percent7.1,
      calculated Upgrade   / calculated TOTAL_CNT as Upgrade_PCT   format=percent7.1,
      calculated OVRD_CNT  / calculated TOTAL_CNT as OVRD_RATE_PCT format=percent7.1,
      sum(case when abs(OVRD_SEVERITY) > 1 then override_ind else 0 end) as OVRD_1NOTCH_CNT format=comma15.0,
      sum(OVRD_SEVERITY) / calculated TOTAL_CNT   as Avg_OVRD_Magnitude
    from Override_&OGM_Qtr._dedup_&Seg
    group by 1
  ;quit;

  /* =================== Total (all heritage) ======================== */
  proc sql;
    create table OVERRIDE_Sum_tot_&Seg as
    select
      count(*)                                   as TOTAL_CNT format=comma15.0,
      sum(override_ind)                          as OVRD_CNT  format=comma15.0,
      sum(case when OVRD_SEVERITY >  0 then override_ind else 0 end) as Downgrade,
      sum(case when OVRD_SEVERITY <  0 then override_ind else 0 end) as Upgrade,
      calculated Downgrade / calculated TOTAL_CNT as Downgrade_PCT format=percent7.1,
      calculated Upgrade   / calculated TOTAL_CNT as Upgrade_PCT   format=percent7.1,
      calculated OVRD_CNT  / calculated TOTAL_CNT as OVRD_RATE_PCT format=percent7.1,
      sum(case when abs(OVRD_SEVERITY) > 1 then override_ind else 0 end) as OVRD_1NOTCH_CNT format=comma15.0,
      sum(OVRD_SEVERITY) / calculated TOTAL_CNT   as Avg_OVRD_Magnitude
    from Override_&OGM_Qtr._dedup_&Seg
  ;quit;

  /**************** Override Report – Current Quarter *****************/
  title "Total Observations in Override Analysis Window (&seg)";
  proc sql noprint; 
    select TOTAL_CNT into :tot_cnt_or from OVERRIDE_Sum_tot_&Seg; 
  quit; 
  %put NOTE: TOTAL_CNT override window (&seg) = &tot_cnt_or; 
  title;

  /* --------- Macro to emit per-heritage pages (exact pattern) ------ */
  %macro or_report(heritage, legacy_bank, legacy_bank_cd);
    data Override_report_1&legacy_bank_cd._&Seg;
      retain Seg Quarter OVRD_1NOTCH_CNT TOTAL_CNT OVRD_1NOTCH_PCT Avg_OVRD_Magnitude;
      set OVERRIDE_Sum_heritage_&Seg(keep= legacy_bank_new OVRD_1NOTCH_CNT TOTAL_CNT OVRD_1NOTCH_PCT Avg_OVRD_Magnitude);
      where legacy_bank_new in (&legacy_bank.);
      Quarter="&OGM_Qtr";
      Seg=upcase("&Seg.");
      rename 
        OVRD_1NOTCH_CNT=OVRD_1NOTCH_CNT_&legacy_bank_cd.
        TOTAL_CNT      =TOTAL_CNT_&legacy_bank_cd.
        OVRD_1NOTCH_PCT=OVRD_1NOTCH_PCT_&legacy_bank_cd.
        Avg_OVRD_Magnitude=AVG_OVRD_Magnitude_&legacy_bank_cd.;
    run;
  %mend;

  /* Example heritage groups — adjust if your OGM uses different labels */
  /* %or_report(heritage,"'BB&T'","hbbt"); */
  /* %or_report(heritage,"'HSI'","hsti");  */
  /* %or_report(heritage,"'Truist'","ttc");*/

  /* Total (all) block exactly like screenshot */
  data New_OGM.Override_report_&OGM_Qtr._&Seg;
    merge
      /* Override_report_hbbt_&Seg (drop=legacy_bank_new) */
      /* Override_report_hsti_&Seg (drop=legacy_bank_new) */
      /* Override_report_ttc__&Seg (drop=legacy_bank_new) */
      OVERRIDE_Sum_tot_&Seg
    ;
  run;

%mend OR;

/* ========= Run it (same entry points as dev’s macro would call) ======== */
/* Set these dates per quarter when you run each slice, just like OGM:    */
/* Example for Q2 window end: &OGM_Cutoff='30JUN2024'd, SF imp: '01JAN2024'd */
%OR(seg=abl, OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd, imp_dtsf='01JAN2024'd);
%OR(seg=sf , OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd, imp_dtsf='01JAN2024'd);
%OR(seg=all, OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd, imp_dtsf='01JAN2024'd);

/* Repeat for 2024Q3/2024Q4/2025Q1 by changing OGM_Qtr and OGM_Cutoff */
