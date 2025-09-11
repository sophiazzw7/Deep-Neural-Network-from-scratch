/* ===========================================================
   MOD1497 — Override Analysis (corrected & consolidated)
   Material override = |OVRD_SEVERITY| > 1  (i.e., ≥2 notches)
   =========================================================== */
options compress=yes reuse=yes mprint mlogic symbolgen;

libname ogm "/sasdata/mtmrg2/users/G07267/MOD1497/MOD1497_full/MRO" access=readonly;

/* ---------------------------
   0) CONFIG
   --------------------------- */
%let QTR_LIST = 2024Q2 2024Q3 2024Q4 2025Q1;

/* helper: convert 'yyyyQn' -> numeric index yyyy*10+n for ordering */
%macro qtr_idx(q);
  input(substr(&q,1,4),8.)*10 + input(substr(&q,6,1),8.)
%mend;

/* ---------------------------
   1) BASE OVERRIDE ROWS (per quarter)
   - ensures OVRD_SEVERITY populated
   - override_ind, material_1notch (|sev|>1)
   --------------------------- */
data _ABL;
  length quarter $7 seg $3 c_obg $200 c_obl 8 c_obgobl $220;
  length model_lgd_grade_num final_lgd_grade_num 8 OVRD_SEVERITY 8 override_ind 8 material_1notch 8;
  stop;
run;

%macro pull_qtr(qtr);
data ABL_&qtr;
  length quarter $7 seg $3 c_obg $200 c_obgobl $220 material_1notch 8;
  set ogm.OVERRIDE_&qtr._DEDUP_ABL
      (keep=c_obg c_obl c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num);
  seg     = 'ABL';
  quarter = "&qtr";

  /* backfill severity if grades exist but severity missing */
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num, model_lgd_grade_num)=0
    then OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;

  override_ind   = (OVRD_SEVERITY ne 0);
  material_1notch= (abs(OVRD_SEVERITY) > 1);   /* >=2 notches */
run;

proc append base=_ABL data=ABL_&qtr force; run;
proc datasets lib=work nolist; delete ABL_&qtr; quit;
%mend;

%macro build_base;
%local i this;
%let i=1;
%do %while(%scan(&QTR_LIST,&i,%str( )) ne );
  %let this=%scan(&QTR_LIST,&i,%str( ));
  %pull_qtr(&this)
  %let i=%eval(&i+1);
%end;
%mend;
%build_base

/* quarter ordering index */
data _ABL; set _ABL; qtr_idx=%qtr_idx(quarter); run;

/* ---------------------------
   2) ATTRIBUTES (face/bal, heritage/LOB/reason)
   Joined by facility key c_obgobl
   --------------------------- */
data _attrs;
  length quarter $7 c_obg $200 c_obl 8 c_obgobl $220
         legacy_bank_new $100 business_group $100
         STI_lob_nm $400 STI_sub_lob_nm $400
         Override_Reason $200 STI_rating_model $200 LASFACILITYNBR $200
         TFC_Curr_Bal 8 tfc_face_amt 8;
  stop;
run;

%macro pull_attrs(qtr);
data attrs_&qtr;
  length quarter $7;
  set ogm.OVERRIDE_&qtr._DEDUP_ABL
      (keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt
            legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
            Override_Reason STI_rating_model LASFACILITYNBR);
  quarter="&qtr";
  Override_Reason=strip(Override_Reason);
run;
proc append base=_attrs data=attrs_&qtr force; run;
proc datasets lib=work nolist; delete attrs_&qtr; quit;
%mend;

%macro build_attrs;
%local i this;
%let i=1;
%do %while(%scan(&QTR_LIST,&i,%str( )) ne );
  %let this=%scan(&QTR_LIST,&i,%str( ));
  %pull_attrs(&this)
  %let i=%eval(&i+1);
%end;
%mend;
%build_attrs

/* enrich base with attributes at obligation level */
proc sort data=_ABL;   by quarter c_obgobl; run;
proc sort data=_attrs; by quarter c_obgobl; run;

proc sql;
create table _abl_enriched as
select a.*,
       b.TFC_Curr_Bal, b.tfc_face_amt,
       b.legacy_bank_new, b.business_group,
       b.STI_lob_nm, b.STI_sub_lob_nm,
       b.Override_Reason, b.STI_rating_model, b.LASFACILITYNBR
from _ABL as a
left join _attrs as b
  on a.quarter=b.quarter and a.c_obgobl=b.c_obgobl;
quit;

/* ---------------------------
   3) REPEAT FLAGS (obligor-level history)
   Uses c_obg consistently
   --------------------------- */
proc sort data=_abl_enriched out=_sorted; by c_obg qtr_idx; run;

data _flags;
  set _sorted;
  by c_obg qtr_idx;
  retain prior_mat prior_any 0;
  if first.c_obg then do; prior_mat=0; prior_any=0; end;

  repeat_material = (material_1notch=1 and prior_mat>0);
  repeat_any      = (override_ind=1   and prior_any>0);

  if material_1notch=1 then prior_mat+1;
  if override_ind=1     then prior_any+1;
run;

/* ---------------------------
   4) CORE TABLES FOR PLOTS
   --------------------------- */

/* 4a. Repeat rate + first-time share */
proc sql;
create table abl_material_repeat_by_qtr as
select quarter,
       count(*)                                     as total_observations,
       sum(material_1notch=1)                       as mat_total,
       sum(repeat_material=1)                       as mat_repeat,
       calculated mat_repeat / max(calculated mat_total,1)      as mat_repeat_pct format=percent8.2,
       sum(material_1notch=1 and repeat_material=0) as mat_first_time,
       calculated mat_first_time / max(calculated mat_total,1)  as mat_first_time_pct format=percent8.2
from _flags
group by quarter
order by quarter;
quit;

/* 4b. Inflation: obligations vs unique obligors */
proc sql;
create table abl_inflation_qtr as
select quarter,
       sum(material_1notch=1)                                     as mat_obligations,
       count(distinct case when material_1notch=1 then c_obg end) as mat_obligors,
       calculated mat_obligations - calculated mat_obligors       as extra_obligations,
       calculated mat_obligations / max(calculated mat_obligors,1)as obligs_per_obligor format=8.2
from _abl_enriched
group by quarter
order by quarter;
quit;

/* 4c. Exposure share: face & balance */
proc sql;
create table abl_exposure_share as
select quarter,
       sum(tfc_face_amt)                                              as face_total,
       sum(case when material_1notch=1 then tfc_face_amt else 0 end)  as face_material,
       calculated face_material / max(calculated face_total,1)        as face_mat_share format=percent8.2,

       sum(TFC_Curr_Bal)                                              as bal_total,
       sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end)  as bal_material,
       calculated bal_material / max(calculated bal_total,1)          as bal_mat_share format=percent8.2
from _abl_enriched
group by quarter
order by quarter;
quit;

/* 4d. Direction: upgrades vs downgrades (reconcile to mat_total) */
proc sql;
create table direction_consistent as
select quarter,
       sum(override_ind=1)                           as overrides,
       sum(material_1notch=1 and OVRD_SEVERITY < 0)  as mat_downgrades,
       sum(material_1notch=1 and OVRD_SEVERITY > 0)  as mat_upgrades,
       sum(material_1notch=1)                        as mat_total
from _abl_enriched
group by quarter
order by quarter;
quit;

/* 4e. Reasons: Top-10 per quarter (rate = material / overrides) */
proc sql;
create table abl_by_reason as
select quarter,
       coalescec(strip(Override_Reason),'(missing)') as reason length=64,
       count(*)                                      as obs,
       sum(override_ind=1)                           as ovr_cnt,
       sum(material_1notch=1)                        as mat_cnt,
       case when calculated ovr_cnt>0
            then calculated mat_cnt / calculated ovr_cnt else . end
                                                     as mat_rate format=percent8.2
from _abl_enriched
group by quarter, calculated reason
order by quarter, mat_cnt desc;
quit;

proc sort data=abl_by_reason out=_r_sorted; by quarter descending mat_cnt; run;

data top10_reasons_per_qtr;
  set _r_sorted;
  by quarter;
  retain _rank;
  if first.quarter then _rank=0;
  _rank+1;
  if _rank<=10;
  drop _rank;
run;

/* 4f. Reason stickiness among repeats (consecutive qtrs, same obligor) */
proc sort data=_abl_enriched out=_mat_only; 
  by c_obg qtr_idx;
  where material_1notch=1;
run;

proc sort data=_flags; by c_obg qtr_idx; run;

data _reason_seq;
  merge _mat_only(in=a keep=quarter qtr_idx c_obg Override_Reason)
        _flags    (in=b keep=c_obg qtr_idx repeat_material);
  by c_obg qtr_idx;
  if a;
  reason = coalescec(strip(Override_Reason),'(missing)');
run;

data _reason_chain;
  set _reason_seq;
  by c_obg qtr_idx;
  lag_reason = lag(reason);
  lag_qtr    = lag(qtr_idx);
  lag_obg    = lag(c_obg);
  same_chain       = (c_obg=lag_obg and qtr_idx = lag_qtr+1);
  same_reason_flag = (same_chain and reason = lag_reason);
run;

proc sql;
create table abl_reason_stickiness as
select quarter,
       count(*)                              as mat_total,
       sum(repeat_material=1)                as mat_repeat,
       sum(same_reason_flag=1)               as same_reason,
       case when calculated mat_repeat>0
            then calculated same_reason / calculated mat_repeat else . end
                                             as same_reason_pct format=percent8.2
from _reason_chain
group by quarter
order by quarter;
quit;

/* 4g. Top repeating obligors by material balance (exposure-focused triage) */
proc sql;
create table top_repeating_obligors_by_balance as
select c_obg,
       sum(repeat_material=1)                                     as repeat_qtrs,
       count(distinct case when material_1notch=1 then quarter end) as material_qtrs,
       sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end) as bal_material,
       sum(case when material_1notch=1 then tfc_face_amt else 0 end)  as face_material,
       catx(',', of quarter)                                       as quarters length=200
from _flags as f
left join _abl_enriched as e
  on f.quarter=e.quarter and f.c_obg=e.c_obg and f.c_obgobl=e.c_obgobl
group by c_obg
having repeat_qtrs>=1
order by bal_material desc;
quit;

/* 4h. Repeat share by heritage & LOB (where repeats concentrate) */
proc sql;
create table repeat_share_by_heritage as
select quarter, legacy_bank_new as heritage,
       sum(repeat_material=1)                        as rpt_total,
       case when sum(material_1notch=1)>0
            then sum(repeat_material=1)/sum(material_1notch=1)
            else . end                               as rpt_share format=percent8.2
from _flags f
left join _abl_enriched e
  on f.quarter=e.quarter and f.c_obgobl=e.c_obgobl
group by quarter, heritage
order by quarter, rpt_share desc;
quit;

proc sql;
create table repeat_share_by_lob as
select quarter, coalescec(STI_lob_nm,'(missing)') as lob,
       sum(repeat_material=1)                        as rpt_total,
       case when sum(material_1notch=1)>0
            then sum(repeat_material=1)/sum(material_1notch=1)
            else . end                               as rpt_share format=percent8.2
from _flags f
left join _abl_enriched e
  on f.quarter=e.quarter and f.c_obgobl=e.c_obgobl
group by quarter, lob
order by quarter, rpt_share desc;
quit;

/* ---------------------------
   5) OPTIONAL sanity checks
   --------------------------- */
proc sql;
title "Reason sums vs Core totals (should match per quarter)";
select a.quarter,
       sum(a.ovr_cnt) as ovr_in_reason,
       sum(a.mat_cnt) as mat_in_reason,
       b.overrides    as ovr_core,
       b.mat_total    as mat_core
from abl_by_reason a
left join direction_consistent b
  on a.quarter=b.quarter
group by a.quarter, b.overrides, b.mat_total
order by a.quarter;
quit;
title;

/* END */
