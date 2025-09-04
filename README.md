/* ============================================================= */
/* Repeats & Counts by Quarter (OGM-aligned, corrected)          */
/*  - ABL window: last-12 months inclusive via upld_dt           */
/*  - SF  window: imp_dtsf .. cutoff inclusive                   */
/*  - Exclude operational reasons ONLY for material calc         */
/*  - De-dup: LAST per (c_obgobl, upld_dt); keep missing dates   */
/*  - Material override = abs(OVRD_SEVERITY) > 1                 */
/*  - Outputs: quarter_summary, global_share                     */
/* ============================================================= */

options compress=yes reuse=yes;

/* ---------- Repos ---------- */
%let ROOT=/sasdata/mrmg1/MOD1497;
libname q2  "&ROOT/2024Q2_v2.1";
libname q3  "&ROOT/2024Q3_v2.1";
libname q4  "&ROOT/2024Q4_v2.1";
libname q1  "&ROOT/2025Q1_v2.2";

libname NEW_OGM work;

/* ---------- LGD letter -> number ---------- */
proc format;
  value $lgd_num
    'A'=1  'B'=2  'C'=3  'D'=4  'E'=5  'F'=6  'G'=7  'H'=8  'I'=9  'J'=10
    'K'=11 'L'=12 'M'=13 'N'=14 'O'=15 'P'=16 'Q'=17 'R'=18 'S'=19 'T'=20;
run;

/* ---------- Stack + alias layer ---------- */
data NEW_OGM.all_table_6ymm_1;
  length OGM_Qtr $6 Seg $2 Override_Reason $30 c_obgobl $200;
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

  /* Reason alias (character) */
  if missing(Override_Reason) then Override_Reason = strip(model_or_reason);

  /* Obligor key alias */
  if not missing(c_obgobl) then c_obgobl = strip(c_obgobl);
  else if not missing(c_obg_obl) then c_obgobl = strip(c_obg_obl);
  else c_obgobl = cats(c_obg,'_',c_obl);

  /* Face amount alias if needed */
  if nmiss(tfc_face_amt)=1 and not missing(face_amt_mtd1) then tfc_face_amt=face_amt_mtd1;
run;

/* ---------- Core OGM slice per quarter/segment ---------- */
%macro OR(seg=abl, OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd, ymm=6ymm, imp_dtsf='01JAN2024'd);

  %local start12;
  %let start12=%sysfunc(intnx(month,&OGM_Cutoff,-11,BEGIN));  /* inclusive lower bound */

  /* ------------------- Build base slice (no op-reason drop) ------------------- */
  data Override_&OGM_Qtr._&Seg;
    set NEW_OGM.all_table_&ymm._1;
    where OGM_Qtr="&OGM_Qtr";

    /* Normalize upload to DATE (handles datetime or date) */
    upld_dt = coalesce(datepart(f_uplddt), f_uplddt);
    format upld_dt date9.;

    /* Base filters */
    if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;

    /* Time window */
    %if %upcase(&seg)=SF %then %do;
      if &imp_dtsf <= upld_dt <= &OGM_Cutoff;
    %end;
    %else %do;
      if &start12  <= upld_dt <= &OGM_Cutoff;
    %end;

    /* LOB + model-grade universe */
    if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
       model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');

    /* Segment flag & severity */
    SF = (upcase("&seg.")='SF');
    model_lgd_grade_num = 1*put(model_lgd_grade,$lgd_num.);
    final_lgd_grade_num = 1*put(final_lgd_grade,$lgd_num.);
    OVRD_SEVERITY       = final_lgd_grade_num - model_lgd_grade_num;  /* + = downgrade */
  run;

  /* ------------------- De-dup: LAST per (c_obgobl, upld_dt) ------------------- */
  data _or_src_;
    set Override_&OGM_Qtr._&Seg;
    _obs = _N_;  /* deterministic tie-breaker */
  run;

  /* Keep missing dates as-is; dedup only when upld_dt exists */
  data _has_dt _no_dt;
    set _or_src_;
    if missing(upld_dt) then output _no_dt;
    else output _has_dt;
  run;

  proc sort data=_has_dt out=_has_dt_srt;
    by c_obgobl upld_dt _obs;
  run;

  data _has_dt_dedup;
    set _has_dt_srt;
    by c_obgobl upld_dt;
    if last.upld_dt;
  run;

  /* Base population AFTER de-dup (use this for "Total Observations") */
  data Override_&OGM_Qtr._dedup_&Seg;
    set _has_dt_dedup _no_dt;
    drop _obs;
  run;

  /* ------------------- Material overrides (now drop op reasons) ------------------- */
  data Override_&OGM_Qtr._mat_&Seg;
    set Override_&OGM_Qtr._dedup_&Seg;
    /* Exclude operational override reasons for material calc */
    if Override_Reason in ('3','4','5','6','7','32','33','36','37','38') then delete;
    if abs(OVRD_SEVERITY) > 1;   /* material */
  run;

%mend OR;

/* ---------- Build quarters (ABL/SF + ALL) ---------- */
%macro build_all_quarters;
  /* 2024Q2 */
  %OR(seg=abl, OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd);
  %OR(seg=sf , OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd);
  data Override_2024Q2_dedup_all; set Override_2024Q2_dedup_abl Override_2024Q2_dedup_sf; OGM_Qtr='2024Q2'; run;
  data Override_2024Q2_mat_all;   set Override_2024Q2_mat_abl   Override_2024Q2_mat_sf;   OGM_Qtr='2024Q2'; run;

  /* 2024Q3 */
  %OR(seg=abl, OGM_Qtr=2024Q3, OGM_Cutoff='30SEP2024'd);
  %OR(seg=sf , OGM_Qtr=2024Q3, OGM_Cutoff='30SEP2024'd);
  data Override_2024Q3_dedup_all; set Override_2024Q3_dedup_abl Override_2024Q3_dedup_sf; OGM_Qtr='2024Q3'; run;
  data Override_2024Q3_mat_all;   set Override_2024Q3_mat_abl   Override_2024Q3_mat_sf;   OGM_Qtr='2024Q3'; run;

  /* 2024Q4 */
  %OR(seg=abl, OGM_Qtr=2024Q4, OGM_Cutoff='31DEC2024'd);
  %OR(seg=sf , OGM_Qtr=2024Q4, OGM_Cutoff='31DEC2024'd);
  data Override_2024Q4_dedup_all; set Override_2024Q4_dedup_abl Override_2024Q4_dedup_sf; OGM_Qtr='2024Q4'; run;
  data Override_2024Q4_mat_all;   set Override_2024Q4_mat_abl   Override_2024Q4_mat_sf;   OGM_Qtr='2024Q4'; run;

  /* 2025Q1 */
  %OR(seg=abl, OGM_Qtr=2025Q1, OGM_Cutoff='31MAR2025'd);
  %OR(seg=sf , OGM_Qtr=2025Q1, OGM_Cutoff='31MAR2025'd);
  data Override_2025Q1_dedup_all; set Override_2025Q1_dedup_abl Override_2025Q1_dedup_sf; OGM_Qtr='2025Q1'; run;
  data Override_2025Q1_mat_all;   set Override_2025Q1_mat_abl   Override_2025Q1_mat_sf;   OGM_Qtr='2025Q1'; run;
%mend build_all_quarters;

%build_all_quarters;

/* ---------- Base population & material events across quarters ---------- */
data all_quarters_base;
  set Override_2024Q2_dedup_all
      Override_2024Q3_dedup_all
      Override_2024Q4_dedup_all
      Override_2025Q1_dedup_all;
run;

data mat_or_all;
  set Override_2024Q2_mat_all
      Override_2024Q3_mat_all
      Override_2024Q4_mat_all
      Override_2025Q1_mat_all;
  keep OGM_Qtr c_obgobl SF Override_Reason OVRD_SEVERITY upld_dt legacy_bank_new;
run;

/* ---------- Repeaters (>=2 quarters with material overrides) ---------- */
proc sql;
  create table repeated as
  select c_obgobl, count(distinct OGM_Qtr) as Qtr_Count, count(*) as Mat_OR_Count
  from mat_or_all
  group by c_obgobl
  having calculated Qtr_Count > 1;
quit;

/* ---------- Per-quarter metrics ---------- */
proc sql;
  /* A) Totals after de-dup (base pop) */
  create table by_qtr_all as
  select OGM_Qtr,
         count(*)                 as Total_Obs,
         count(distinct c_obgobl) as Distinct_Obligors
  from all_quarters_base
  group by OGM_Qtr
  order by OGM_Qtr;

  /* B) Material override events & distinct obligors */
  create table by_qtr_mat as
  select OGM_Qtr,
         count(*)                 as Mat_OR_Events,
         count(distinct c_obgobl) as Mat_OR_Obligors
  from mat_or_all
  group by OGM_Qtr
  order by OGM_Qtr;

  /* C) Repeating obligors and events (per quarter) */
  create table _mat_tag as
  select m.*, (case when r.c_obgobl is not null then 1 else 0 end) as Is_Repeater
  from mat_or_all m
  left join repeated r
    on m.c_obgobl = r.c_obgobl;

  create table by_qtr_repeat as
  select OGM_Qtr,
         sum(Is_Repeater)                                       as Repeating_Events,
         count(distinct case when Is_Repeater=1 then c_obgobl end)
                                                                as Repeating_Obligors_GE2Qtrs
  from _mat_tag
  group by OGM_Qtr
  order by OGM_Qtr;
quit;

/* ---------- Final per-quarter table ---------- */
proc sql;
  create table quarter_summary as
  select a.OGM_Qtr,
         a.Total_Obs,
         a.Distinct_Obligors,
         b.Mat_OR_Events,
         b.Mat_OR_Obligors,
         coalesce(c.Repeating_Obligors_GE2Qtrs,0) as Repeating_Obligors_GE2Qtrs,
         coalesce(c.Repeating_Events,0)           as Repeating_Events
  from by_qtr_all a
  left join by_qtr_mat b on a.OGM_Qtr=b.OGM_Qtr
  left join by_qtr_repeat c on a.OGM_Qtr=c.OGM_Qtr
  order by a.OGM_Qtr;
quit;

/* ---------- Display ---------- */
title "Quarterly Summary: totals, material overrides, repeaters";
proc print data=quarter_summary noobs; run;
title;

/* ---------- Global share of events from repeaters ---------- */
proc sql noprint;
  select sum(Mat_OR_Events), sum(Repeating_Events)
    into :tot_mat, :rep_events
  from quarter_summary;
quit;

data global_share;
  Total_Material_Override_Events = &tot_mat.;
  Repeating_Events               = &rep_events.;
  if &tot_mat.>0 then Share_Repeating_Events = &rep_events./&tot_mat.;
  format Share_Repeating_Events percent8.2;
run;

title "Global Share of Material-Override Events from Repeaters";
proc print data=global_share noobs; run;
title;
