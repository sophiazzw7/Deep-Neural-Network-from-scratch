/* ============================================================= */
/*   REPEATED MATERIAL OVERRIDES (4 QUARTERS, ABL+SF, ALL)       */
/*   - builds per-quarter, OGM-style post-filter, de-dup sets    */
/*   - gathers |OVRD_SEVERITY|>1 events                          */
/*   - flags obligors repeating across quarters                  */
/*   - summary + details + optional CSV exports                  */
/* ============================================================= */

options compress=yes reuse=yes;

/* ---------- Assumes you already have these libnames ----------- */
/* %let ROOT=/sasdata/mrmg1/MOD1497;
libname q2  "&ROOT/2024Q2_v2.1";
libname q3  "&ROOT/2024Q3_v2.1";
libname q4  "&ROOT/2024Q4_v2.1";
libname q1  "&ROOT/2025Q1_v2.2"; */

libname New_OGM work;

/* ---------- Formats: LGD letter -> numeric -------------------- */
proc format;
  value $lgd_num
    'A'=1  'B'=2  'C'=3  'D'=4  'E'=5  'F'=6  'G'=7  'H'=8  'I'=9  'J'=10
    'K'=11 'L'=12 'M'=13 'N'=14 'O'=15 'P'=16 'Q'=17 'R'=18 'S'=19 'T'=20
  ;
run;

/* ---------- Canonical stacked alias layer (same as before) ---- */
data New_OGM.all_table_6ymm_1;
  length OGM_Qtr $6 Seg $2 Override_Reason $10 c_obgobl $200;
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

  /* Aliases */
  if missing(Override_Reason) then Override_Reason = strip(model_or_reason);

  if not missing(c_obgobl) then c_obgobl=strip(c_obgobl);
  else if not missing(c_obg_obl) then c_obgobl=strip(c_obg_obl);
  else c_obgobl = cats(c_obg,'_',c_obl);

  if nmiss(tfc_face_amt)=1 and not missing(face_amt_mtd1) then tfc_face_amt=face_amt_mtd1;
run;

/* ---------- Core macro: build OGM-style post-filter set -------- */
%macro OR(seg=abl, OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd, ymm=6ymm, imp_dtsf='01JAN2024'd);

  %local OR_window_stat;
  %let OR_window_stat=%sysfunc(intnx(month,&OGM_Cutoff,-11,BEG));

  data Override_&OGM_Qtr._&Seg;
    set New_OGM.all_table_&ymm._1;
    where OGM_Qtr="&OGM_Qtr";

    /* Base filters */
    if model_lgd_grade^='' and final_lgd_grade^='' and tfc_face_amt>0;

    /* Time window: SF uses imp_dtsf lower bound, otherwise last-12-mo via f_uplddt */
    %if &seg.=sf %then %do;
      if &imp_dtsf <= f_uplddt <= &OGM_Cutoff;
    %end;
    %else %do;
      if &OR_window_stat < f_uplddt <= &OGM_Cutoff;
    %end;

    /* LOB + grade universe */
    if STI_lob_NM not in ('Consumer Credit','Retail Banking') 
       and model_lgd_grade in ('A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R');

    SF = (%sysfunc(upcase(&seg.)) = SF);

    /* Numericize & severity */
    model_lgd_grade_num = 1*put(model_lgd_grade,$lgd_num.);
    final_lgd_grade_num = 1*put(final_lgd_grade,$lgd_num.);
    override_ind        = (final_lgd_grade ne model_lgd_grade);
    OVRD_SEVERITY       = final_lgd_grade_num - model_lgd_grade_num;  /* + = downgrade */

    /* Drop operational override reasons (same list) */
    if Override_Reason in ('3','4','5','6','7','32','33','36','37','38') then delete;
  run;

  /* De-dup by obligor + date (keep last) */
  proc sort data=Override_&OGM_Qtr._&Seg; by c_obgobl f_uplddt; run;

  data Override_&OGM_Qtr._dedup_&Seg;
    set Override_&OGM_Qtr._&Seg;
    by c_obgobl f_uplddt;
    if last.c_obgobl;
  run;

%mend OR;

/* ---------- Build per-quarter, for ABL and SF, then ALL --------- */
%macro build_all_quarters;
  /* 2024Q2 */
  %OR(seg=abl, OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd, imp_dtsf='01JAN2024'd);
  %OR(seg=sf , OGM_Qtr=2024Q2, OGM_Cutoff='30JUN2024'd, imp_dtsf='01JAN2024'd);
  data Override_2024Q2_dedup_all; set Override_2024Q2_dedup_abl Override_2024Q2_dedup_sf; OGM_Qtr='2024Q2'; run;

  /* 2024Q3 */
  %OR(seg=abl, OGM_Qtr=2024Q3, OGM_Cutoff='30SEP2024'd, imp_dtsf='01JAN2024'd);
  %OR(seg=sf , OGM_Qtr=2024Q3, OGM_Cutoff='30SEP2024'd, imp_dtsf='01JAN2024'd);
  data Override_2024Q3_dedup_all; set Override_2024Q3_dedup_abl Override_2024Q3_dedup_sf; OGM_Qtr='2024Q3'; run;

  /* 2024Q4 */
  %OR(seg=abl, OGM_Qtr=2024Q4, OGM_Cutoff='31DEC2024'd, imp_dtsf='01JAN2024'd);
  %OR(seg=sf , OGM_Qtr=2024Q4, OGM_Cutoff='31DEC2024'd, imp_dtsf='01JAN2024'd);
  data Override_2024Q4_dedup_all; set Override_2024Q4_dedup_abl Override_2024Q4_dedup_sf; OGM_Qtr='2024Q4'; run;

  /* 2025Q1 */
  %OR(seg=abl, OGM_Qtr=2025Q1, OGM_Cutoff='31MAR2025'd, imp_dtsf='01JAN2024'd);
  %OR(seg=sf , OGM_Qtr=2025Q1, OGM_Cutoff='31MAR2025'd, imp_dtsf='01JAN2024'd);
  data Override_2025Q1_dedup_all; set Override_2025Q1_dedup_abl Override_2025Q1_dedup_sf; OGM_Qtr='2025Q1'; run;
%mend build_all_quarters;

%build_all_quarters;

/* ---------- Collect MATERIAL overrides across all four quarters -- */
data mat_or_all;
  length OGM_Qtr $6;
  set 
    Override_2024Q2_dedup_all
    Override_2024Q3_dedup_all
    Override_2024Q4_dedup_all
    Override_2025Q1_dedup_all
  ;
  mat_ovrd = (abs(OVRD_SEVERITY) > 1);
  if mat_ovrd;           /* keep only material overrides */
  keep OGM_Qtr c_obgobl SF Override_Reason OVRD_SEVERITY f_uplddt legacy_bank_new;
run;

/* ---------- Repeated obligors across quarters -------------------- */
proc sql;
  /* How many quarters each obligor appears in with a material override */
  create table repeated as
  select 
      c_obgobl,
      count(distinct OGM_Qtr) as Qtr_Count,
      count(*)                 as Mat_OR_Count
  from mat_or_all
  group by c_obgobl
  having calculated Qtr_Count > 1
  ;

  /* Global summary */
  create table rep_summary as
  select 
      (select count(*) from mat_or_all)              as Total_Material_Overrides,
      (select count(distinct c_obgobl) from mat_or_all) as Distinct_Obligors_with_MatOR,
      (select count(*) from repeated)                as Repeated_Obligors_GE2Qtrs,
      (calculated Repeated_Obligors_GE2Qtrs) 
          / (select count(*) from mat_or_all)        as Share_of_All_MatOR format=percent8.2
  ;
quit;

/* ---------- Helpful breakouts ----------------------------------- */
/* By quarter: distinct obligors with material override */
proc sql;
  create table by_quarter as
  select OGM_Qtr, 
         count(*) as Mat_OR_Events,
         count(distinct c_obgobl) as Mat_OR_Obligors
  from mat_or_all
  group by OGM_Qtr
  order by OGM_Qtr;
quit;

/* By segment (ABL vs SF) */
proc sql;
  create table by_seg as
  select (case when SF=1 then 'SF' else 'ABL' end) as Segment length=3,
         count(*) as Mat_OR_Events,
         count(distinct c_obgobl) as Mat_OR_Obligors
  from mat_or_all
  group by Segment;
quit;

/* Top repeaters (who, how many quarters, how many events) */
proc sql;
  create table top_repeaters as
  select r.c_obgobl, r.Qtr_Count, r.Mat_OR_Count,
         /* optional: list the quarters as a comma string */
         catsx(',', of qtrs[*]) as Quarter_List
  from repeated r
  left join (
      select c_obgobl,
             /* Build wide flags then catsx them; simpler approach below */
             strip(prxchange('s/\s+/,/o', -1, catx(' ',
                max((OGM_Qtr='2024Q2')*'2024Q2'),
                max((OGM_Qtr='2024Q3')*'2024Q3'),
                max((OGM_Qtr='2024Q4')*'2024Q4'),
                max((OGM_Qtr='2025Q1')*'2025Q1')
             ))) as qtrs
      from mat_or_all
      group by c_obgobl
  ) q on r.c_obgobl=q.c_obgobl
  order by r.Qtr_Count desc, r.Mat_OR_Count desc
  ;
quit;

/* ---------- (Optional) export CSVs for memo/tableau -------------- */
/* filename outdir "/sasdata/mrmg1/MOD1497/exports";  */
/* proc export data=rep_summary    outfile=outdir||"/rep_summary.csv" dbms=csv replace; run; */
/* proc export data=by_quarter     outfile=outdir||"/rep_by_quarter.csv" dbms=csv replace; run; */
/* proc export data=by_seg         outfile=outdir||"/rep_by_segment.csv" dbms=csv replace; run; */
/* proc export data=top_repeaters  outfile=outdir||"/rep_top_repeaters.csv" dbms=csv replace; run; */

/* ---------- Quick viewer ---------------------------------------- */
title "Repeated Material Overrides – Global Summary";
proc print data=rep_summary noobs; run;

title "Material Overrides by Quarter";
proc print data=by_quarter noobs; run;

title "Material Overrides by Segment (ABL vs SF)";
proc print data=by_seg noobs; run;

title "Top Repeat Obligors (>=2 quarters)";
proc print data=top_repeaters(obs=50) noobs; run;
title;
