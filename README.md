/* =================== SETUP =================== */
options mprint mlogic symbolgen msglevel=i;
ods listing; ods graphics off;

%let ROOT=/sasdata/mrmg1/MOD1497;   /* <— change if your path differs */
libname q2  "&ROOT/2024Q2_v2.1";
libname q3  "&ROOT/2024Q3_v2.1";
libname q4  "&ROOT/2024Q4_v2.1";
libname q1  "&ROOT/2025Q1_v2.2";
libname NEW_OGM work;

/* LGD letter -> number */
proc format;
  value $lgd_num
    'A'=1 'B'=2 'C'=3 'D'=4 'E'=5 'F'=6 'G'=7 'H'=8 'I'=9 'J'=10
    'K'=11 'L'=12 'M'=13 'N'=14 'O'=15 'P'=16 'Q'=17 'R'=18 'S'=19 'T'=20;
run;

/* =================== STACK & ALIASES =================== */
data NEW_OGM.all_table_6ymm_1;
  length OGM_Qtr $6 Seg $2 c_obgobl $200 Override_Reason_c $30;
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

  /* quarter tag */
  if a1 or a2 then OGM_Qtr='2024Q2';
  else if b1 or b2 then OGM_Qtr='2024Q3';
  else if c1 or c2 then OGM_Qtr='2024Q4';
  else if d1 or d2 then OGM_Qtr='2025Q1';

  /* segment tag */
  Seg = ifc(a2 or b2 or c2 or d2,'sf','abl');

  /* obligor key */
  if not missing(c_obgobl) then c_obgobl=strip(c_obgobl);
  else if not missing(c_obg_obl) then c_obgobl=strip(c_obg_obl);
  else c_obgobl=cats(coalescec(c_obg,'?'), '_', coalescec(c_obl,'?'));

  /* reason -> character; use model_or_reason if present */
  if ^missing(model_or_reason) then Override_Reason_c=strip(vvalue(model_or_reason));
  else if ^missing(Override_Reason) then Override_Reason_c=strip(vvalue(Override_Reason));

  /* face amount alias */
  if missing(tfc_face_amt) and not missing(face_amt_mtd1) then tfc_face_amt=face_amt_mtd1;
run;

proc sql; select count(*) as rows_after_stack from NEW_OGM.all_table_6ymm_1; quit;

/* Helper: normalize f_uplddt to a DATE into variable upld_dt (robust to date/datetime/char) */
data NEW_OGM.all_table_6ymm_1;
  set NEW_OGM.all_table_6ymm_1;
  length upld_dt 8; format upld_dt date9.;
  if vtype(f_uplddt)='N' then do;
    upld_dt = datepart(f_uplddt);
    if missing(upld_dt) then upld_dt = f_uplddt;        /* numeric DATE case */
  end;
  else if vtype(f_uplddt)='C' then do;
    length __dt 8;
    __dt = input(f_uplddt, anydtdtm.);
    if not missing(__dt) then upld_dt = datepart(__dt);
    else upld_dt = input(f_uplddt, anydtdte.);
  end;
run;

/* =================== CONSTANT WINDOWS (no macros) =================== */
/* ABL windows (inclusive lower bound): */
%let q2_start = '01JUL2023'd; %let q2_end = '30JUN2024'd;
%let q3_start = '01OCT2023'd; %let q3_end = '30SEP2024'd;
%let q4_start = '01JAN2024'd; %let q4_end = '31DEC2024'd;
%let q1_start = '01APR2024'd; %let q1_end = '31MAR2025'd;

/* SF lower bound (imp_dtsf) used for all quarters to their respective cutoffs */
%let sf_lb = '01JAN2024'd;

/* =================== BUILD PER QUARTER / SEGMENT (no macros) =================== */
/* --- 2024Q2 ABL BASE --- */
data Override_2024Q2_abl_base;
  set NEW_OGM.all_table_6ymm_1(where=(OGM_Qtr='2024Q2' and Seg='abl'));
  if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;
  if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
     model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');
  if &q2_start <= upld_dt <= &q2_end;
  SF = 0;
  model_lgd_grade_num=1*put(model_lgd_grade,$lgd_num.);
  final_lgd_grade_num=1*put(final_lgd_grade,$lgd_num.);
  OVRD_SEVERITY=final_lgd_grade_num - model_lgd_grade_num;
run;

/* --- 2024Q2 ABL DEDUP (LAST per c_obgobl,upld_dt; keep missing dates) --- */
data _src; set Override_2024Q2_abl_base; _obs=_n_; run;
data _has _nod; set _src; if missing(upld_dt) then output _nod; else output _has; run;
proc sort data=_has out=_has_s; by c_obgobl upld_dt _obs; run;
data _has_d; set _has_s; by c_obgobl upld_dt; if last.upld_dt; run;
data Override_2024Q2_dedup_abl; set _has_d _nod; drop _obs; run;
proc datasets lib=work nolist; delete _src _has _nod _has_s _has_d; quit;

/* --- 2024Q2 SF BASE --- */
data Override_2024Q2_sf_base;
  set NEW_OGM.all_table_6ymm_1(where=(OGM_Qtr='2024Q2' and Seg='sf'));
  if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;
  if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
     model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');
  if &sf_lb <= upld_dt <= &q2_end;
  SF = 1;
  model_lgd_grade_num=1*put(model_lgd_grade,$lgd_num.);
  final_lgd_grade_num=1*put(final_lgd_grade,$lgd_num.);
  OVRD_SEVERITY=final_lgd_grade_num - model_lgd_grade_num;
run;

/* --- 2024Q2 SF DEDUP --- */
data _src; set Override_2024Q2_sf_base; _obs=_n_; run;
data _has _nod; set _src; if missing(upld_dt) then output _nod; else output _has; run;
proc sort data=_has out=_has_s; by c_obgobl upld_dt _obs; run;
data _has_d; set _has_s; by c_obgobl upld_dt; if last.upld_dt; run;
data Override_2024Q2_dedup_sf; set _has_d _nod; drop _obs; run;
proc datasets lib=work nolist; delete _src _has _nod _has_s _has_d; quit;

/* --- 2024Q2 UNION + MATERIAL (drop operational reasons here only) --- */
data Override_2024Q2_dedup_all; set Override_2024Q2_dedup_abl Override_2024Q2_dedup_sf; OGM_Qtr='2024Q2'; run;
data Override_2024Q2_mat_all;
  set Override_2024Q2_dedup_all;
  if Override_Reason_c in ('3','4','5','6','7','32','33','36','37','38') then delete;
  if abs(OVRD_SEVERITY) > 1;
  OGM_Qtr='2024Q2';
run;

proc sql;
  select '2024Q2_dedup_all' as ds, count(*) as n from Override_2024Q2_dedup_all
  union all
  select '2024Q2_mat_all'  , count(*)       from Override_2024Q2_mat_all;
quit;

/* ===== Repeat the same pattern for 2024Q3 / 2024Q4 / 2025Q1 ===== */

/* --- 2024Q3 ABL --- */
data Override_2024Q3_abl_base;
  set NEW_OGM.all_table_6ymm_1(where=(OGM_Qtr='2024Q3' and Seg='abl'));
  if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;
  if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
     model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');
  if &q3_start <= upld_dt <= &q3_end;
  SF=0;
  model_lgd_grade_num=1*put(model_lgd_grade,$lgd_num.);
  final_lgd_grade_num=1*put(final_lgd_grade,$lgd_num.);
  OVRD_SEVERITY=final_lgd_grade_num - model_lgd_grade_num;
run;
data _src; set Override_2024Q3_abl_base; _obs=_n_; run;
data _has _nod; set _src; if missing(upld_dt) then output _nod; else output _has; run;
proc sort data=_has out=_has_s; by c_obgobl upld_dt _obs; run;
data _has_d; set _has_s; by c_obgobl upld_dt; if last.upld_dt; run;
data Override_2024Q3_dedup_abl; set _has_d _nod; drop _obs; run;
proc datasets lib=work nolist; delete _src _has _nod _has_s _has_d; quit;

/* --- 2024Q3 SF --- */
data Override_2024Q3_sf_base;
  set NEW_OGM.all_table_6ymm_1(where=(OGM_Qtr='2024Q3' and Seg='sf'));
  if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;
  if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
     model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');
  if &sf_lb <= upld_dt <= &q3_end; SF=1;
  model_lgd_grade_num=1*put(model_lgd_grade,$lgd_num.);
  final_lgd_grade_num=1*put(final_lgd_grade,$lgd_num.);
  OVRD_SEVERITY=final_lgd_grade_num - model_lgd_grade_num;
run;
data _src; set Override_2024Q3_sf_base; _obs=_n_; run;
data _has _nod; set _src; if missing(upld_dt) then output _nod; else output _has; run;
proc sort data=_has out=_has_s; by c_obgobl upld_dt _obs; run;
data _has_d; set _has_s; by c_obgobl upld_dt; if last.upld_dt; run;
data Override_2024Q3_dedup_sf; set _has_d _nod; drop _obs; run;
proc datasets lib=work nolist; delete _src _has _nod _has_s _has_d; quit;

data Override_2024Q3_dedup_all; set Override_2024Q3_dedup_abl Override_2024Q3_dedup_sf; OGM_Qtr='2024Q3'; run;
data Override_2024Q3_mat_all;
  set Override_2024Q3_dedup_all;
  if Override_Reason_c in ('3','4','5','6','7','32','33','36','37','38') then delete;
  if abs(OVRD_SEVERITY) > 1;
  OGM_Qtr='2024Q3';
run;

proc sql;
  select '2024Q3_dedup_all' as ds, count(*) as n from Override_2024Q3_dedup_all
  union all
  select '2024Q3_mat_all'  , count(*)       from Override_2024Q3_mat_all;
quit;

/* --- 2024Q4 --- */
data Override_2024Q4_abl_base;
  set NEW_OGM.all_table_6ymm_1(where=(OGM_Qtr='2024Q4' and Seg='abl'));
  if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;
  if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
     model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');
  if &q4_start <= upld_dt <= &q4_end; SF=0;
  model_lgd_grade_num=1*put(model_lgd_grade,$lgd_num.);
  final_lgd_grade_num=1*put(final_lgd_grade,$lgd_num.);
  OVRD_SEVERITY=final_lgd_grade_num - model_lgd_grade_num;
run;
data _src; set Override_2024Q4_abl_base; _obs=_n_; run;
data _has _nod; set _src; if missing(upld_dt) then output _nod; else output _has; run;
proc sort data=_has out=_has_s; by c_obgobl upld_dt _obs; run;
data _has_d; set _has_s; by c_obgobl upld_dt; if last.upld_dt; run;
data Override_2024Q4_dedup_abl; set _has_d _nod; drop _obs; run;
proc datasets lib=work nolist; delete _src _has _nod _has_s _has_d; quit;

data Override_2024Q4_sf_base;
  set NEW_OGM.all_table_6ymm_1(where=(OGM_Qtr='2024Q4' and Seg='sf'));
  if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;
  if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
     model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');
  if &sf_lb <= upld_dt <= &q4_end; SF=1;
  model_lgd_grade_num=1*put(model_lgd_grade,$lgd_num.);
  final_lgd_grade_num=1*put(final_lgd_grade,$lgd_num.);
  OVRD_SEVERITY=final_lgd_grade_num - model_lgd_grade_num;
run;
data _src; set Override_2024Q4_sf_base; _obs=_n_; run;
data _has _nod; set _src; if missing(upld_dt) then output _nod; else output _has; run;
proc sort data=_has out=_has_s; by c_obgobl upld_dt _obs; run;
data _has_d; set _has_s; by c_obgobl upld_dt; if last.upld_dt; run;
data Override_2024Q4_dedup_sf; set _has_d _nod; drop _obs; run;
proc datasets lib=work nolist; delete _src _has _nod _has_s _has_d; quit;

data Override_2024Q4_dedup_all; set Override_2024Q4_dedup_abl Override_2024Q4_dedup_sf; OGM_Qtr='2024Q4'; run;
data Override_2024Q4_mat_all;
  set Override_2024Q4_dedup_all;
  if Override_Reason_c in ('3','4','5','6','7','32','33','36','37','38') then delete;
  if abs(OVRD_SEVERITY) > 1;
  OGM_Qtr='2024Q4';
run;

proc sql;
  select '2024Q4_dedup_all' as ds, count(*) as n from Override_2024Q4_dedup_all
  union all
  select '2024Q4_mat_all'  , count(*)       from Override_2024Q4_mat_all;
quit;

/* --- 2025Q1 --- */
data Override_2025Q1_abl_base;
  set NEW_OGM.all_table_6ymm_1(where=(OGM_Qtr='2025Q1' and Seg='abl'));
  if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;
  if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
     model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');
  if &q1_start <= upld_dt <= &q1_end; SF=0;
  model_lgd_grade_num=1*put(model_lgd_grade,$lgd_num.);
  final_lgd_grade_num=1*put(final_lgd_grade,$lgd_num.);
  OVRD_SEVERITY=final_lgd_grade_num - model_lgd_grade_num;
run;
data _src; set Override_2025Q1_abl_base; _obs=_n_; run;
data _has _nod; set _src; if missing(upld_dt) then output _nod; else output _has; run;
proc sort data=_has out=_has_s; by c_obgobl upld_dt _obs; run;
data _has_d; set _has_s; by c_obgobl upld_dt; if last.upld_dt; run;
data Override_2025Q1_dedup_abl; set _has_d _nod; drop _obs; run;
proc datasets lib=work nolist; delete _src _has _nod _has_s _has_d; quit;

data Override_2025Q1_sf_base;
  set NEW_OGM.all_table_6ymm_1(where=(OGM_Qtr='2025Q1' and Seg='sf'));
  if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;
  if STI_lob_NM not in ('Consumer Credit','Retail Banking') and
     model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');
  if &sf_lb <= upld_dt <= &q1_end; SF=1;
  model_lgd_grade_num=1*put(model_lgd_grade,$lgd_num.);
  final_lgd_grade_num=1*put(final_lgd_grade,$lgd_num.);
  OVRD_SEVERITY=final_lgd_grade_num - model_lgd_grade_num;
run;
data _src; set Override_2025Q1_sf_base; _obs=_n_; run;
data _has _nod; set _src; if missing(upld_dt) then output _nod; else output _has; run;
proc sort data=_has out=_has_s; by c_obgobl upld_dt _obs; run;
data _has_d; set _has_s; by c_obgobl upld_dt; if last.upld_dt; run;
data Override_2025Q1_dedup_sf; set _has_d _nod; drop _obs; run;
proc datasets lib=work nolist; delete _src _has _nod _has_s _has_d; quit;

data Override_2025Q1_dedup_all; set Override_2025Q1_dedup_abl Override_2025Q1_dedup_sf; OGM_Qtr='2025Q1'; run;
data Override_2025Q1_mat_all;
  set Override_2025Q1_dedup_all;
  if Override_Reason_c in ('3','4','5','6','7','32','33','36','37','38') then delete;
  if abs(OVRD_SEVERITY) > 1;
  OGM_Qtr='2025Q1';
run;

proc sql;
  select '2025Q1_dedup_all' as ds, count(*) as n from Override_2025Q1_dedup_all
  union all
  select '2025Q1_mat_all'  , count(*)       from Override_2025Q1_mat_all;
quit;

/* =================== ROLLUPS =================== */
data all_quarters_base;
  set Override_2024Q2_dedup_all Override_2024Q3_dedup_all
      Override_2024Q4_dedup_all Override_2025Q1_dedup_all;
run;

data mat_or_all;
  set Override_2024Q2_mat_all Override_2024Q3_mat_all
      Override_2024Q4_mat_all Override_2025Q1_mat_all;
  keep OGM_Qtr c_obgobl SF Override_Reason_c OVRD_SEVERITY upld_dt legacy_bank_new;
run;

/* Repeated obligors (>=2 distinct quarters with material override) */
proc sql;
  create table repeated as
  select c_obgobl,
         count(distinct OGM_Qtr) as Qtr_Count,
         count(*) as Mat_OR_Count
  from mat_or_all
  group by c_obgobl
  having calculated Qtr_Count > 1;
quit;

/* Per-quarter metrics */
proc sql;
  create table by_qtr_all as
  select OGM_Qtr,
         count(*) as Total_Obs,
         count(distinct c_obgobl) as Distinct_Obligors
  from all_quarters_base
  group by OGM_Qtr
  order by OGM_Qtr;

  create table by_qtr_mat as
  select OGM_Qtr,
         count(*) as Mat_OR_Events,
         count(distinct c_obgobl) as Mat_OR_Obligors
  from mat_or_all
  group by OGM_Qtr
  order by OGM_Qtr;

  /* tag repeaters */
  create table _mat_tag as
  select m.*,
         (case when r.c_obgobl is not null then 1 else 0 end) as Is_Repeater
  from mat_or_all m
  left join repeated r
    on m.c_obgobl=r.c_obgobl;

  create table by_qtr_repeat as
  select OGM_Qtr,
         sum(Is_Repeater) as Repeating_Events,
         count(distinct case when Is_Repeater=1 then c_obgobl end)
           as Repeating_Obligors_GE2Qtrs
  from _mat_tag
  group by OGM_Qtr
  order by OGM_Qtr;
quit;

/* Final per-quarter table */
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

/* =================== SHOW RESULTS =================== */
title "Quarterly Summary: totals, material overrides, repeaters";
proc print data=quarter_summary noobs; run; title;

proc sql noprint;
  select sum(Mat_OR_Events), sum(Repeating_Events)
    into :tot_mat, :rep_evt
  from quarter_summary;
quit;

data global_share;
  Total_Material_Override_Events=&tot_mat.;
  Repeating_Events=&rep_evt.;
  if &tot_mat.>0 then Share_Repeating_Events=&rep_evt./&tot_mat.;
  format Share_Repeating_Events percent8.2;
run;

title "Global Share of Material-Override Events from Repeaters";
proc print data=global_share noobs; run; title;
