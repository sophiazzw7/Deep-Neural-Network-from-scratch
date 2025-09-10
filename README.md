/* =========  ABL investigation add-ons (paste after abl_material_repeat_by_qtr)  ========= */
/* Uses libname ogm already defined earlier. If not, re-define it here. */

/* -- A) Pull attributes from the saved developer DEDUP tables and join to your _ABL -- */
data _attrs_q2;
  length quarter $7 c_obgobl $200 legacy_bank_new $100 business_group $100
         STI_lob_nm $400 STI_sub_lob_nm $400 TFC_src_sys_cd $50 STI_rating_model $50
         LASFACILITYNBR $100;
  set ogm.OVERRIDE_2024Q2_DEDUP_ABL
      (keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt Override_Reason
            legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
            TFC_src_sys_cd STI_rating_model LASFACILITYNBR);
  quarter="2024Q2";
run;

data _attrs_q3;
  length quarter $7 c_obgobl $200 legacy_bank_new $100 business_group $100
         STI_lob_nm $400 STI_sub_lob_nm $400 TFC_src_sys_cd $50 STI_rating_model $50
         LASFACILITYNBR $100;
  set ogm.OVERRIDE_2024Q3_DEDUP_ABL
      (keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt Override_Reason
            legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
            TFC_src_sys_cd STI_rating_model LASFACILITYNBR);
  quarter="2024Q3";
run;

data _attrs_q4;
  length quarter $7 c_obgobl $200 legacy_bank_new $100 business_group $100
         STI_lob_nm $400 STI_sub_lob_nm $400 TFC_src_sys_cd $50 STI_rating_model $50
         LASFACILITYNBR $100;
  set ogm.OVERRIDE_2024Q4_DEDUP_ABL
      (keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt Override_Reason
            legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
            TFC_src_sys_cd STI_rating_model LASFACILITYNBR);
  quarter="2024Q4";
run;

data _attrs_q1;
  length quarter $7 c_obgobl $200 legacy_bank_new $100 business_group $100
         STI_lob_nm $400 STI_sub_lob_nm $400 TFC_src_sys_cd $50 STI_rating_model $50
         LASFACILITYNBR $100;
  set ogm.OVERRIDE_2025Q1_DEDUP_ABL
      (keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt Override_Reason
            legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
            TFC_src_sys_cd STI_rating_model LASFACILITYNBR);
  quarter="2025Q1";
run;

data _ABL_attrs; set _attrs_q2 _attrs_q3 _attrs_q4 _attrs_q1; run;
proc sort data=_ABL_attrs nodupkey; by quarter c_obgobl; run;

/* Join attributes onto your obligation-level _ABL table */
proc sql;
  create table _ABL_enriched as
  select a.*, 
         b.c_obg, b.c_obl, b.TFC_Curr_Bal, b.tfc_face_amt, b.Override_Reason,
         b.legacy_bank_new, b.business_group, b.STI_lob_nm, b.STI_sub_lob_nm,
         b.TFC_src_sys_cd, b.STI_rating_model, b.LASFACILITYNBR
  from _ABL as a
  left join _ABL_attrs as b
    on a.quarter=b.quarter and a.c_obgobl=b.c_obgobl;
quit;

/* -- B) Inflation check: obligations vs unique obligors with MATERIAL overrides -- */
proc sql;
  create table abl_material_inflation_qtr as
  select quarter,
         sum(material_1notch)                                          as mat_total_obligations,
         count(distinct case when material_1notch=1 then c_obg end)    as mat_total_obligors,
         calculated mat_total_obligations - calculated mat_total_obligors as extra_obligations,
         calculated mat_total_obligations / max(calculated mat_total_obligors,1)
           as obligations_per_obligor format=8.2
  from _ABL_enriched
  group by quarter
  order by quarter;
quit;

/* Distribution: among obligors with material this quarter, how many obligations carried it? */
proc sql;
  create table abl_mat_obl_per_obg as
  select quarter, c_obg, count(*) as obligations_with_material
  from _ABL_enriched
  where material_1notch=1
  group by quarter, c_obg;
quit;

proc sql;
  create table abl_mat_obl_per_obg_dist as
  select quarter, obligations_with_material, count(*) as obligors
  from abl_mat_obl_per_obg
  group by quarter, obligations_with_material
  order by quarter, obligations_with_material;
quit;

/* -- C) Severity shape and upgrade/downgrade mix (obligation level) -- */
data _ABL_enriched;
  set _ABL_enriched;
  length sev_bucket $6;
  if OVRD_SEVERITY>= 3 then sev_bucket='>=+3';
  else if OVRD_SEVERITY= 2 then sev_bucket='+2';
  else if OVRD_SEVERITY= 1 then sev_bucket='+1';
  else if OVRD_SEVERITY= 0 then sev_bucket='0';
  else if OVRD_SEVERITY=-1 then sev_bucket='-1';
  else if OVRD_SEVERITY=-2 then sev_bucket='-2';
  else if OVRD_SEVERITY<=-3 then sev_bucket='<=-3';
run;

proc sql;
  create table abl_severity_mix as
  select quarter, sev_bucket,
         count(*) as obs,
         sum(override_ind) as ovr_cnt,
         sum(material_1notch) as mat_cnt,
         (calculated mat_cnt / calculated obs) as mat_rate format=percent8.2
  from _ABL_enriched
  group by quarter, sev_bucket
  order by quarter, sev_bucket;
quit;

proc sql;
  create table abl_updown as
  select quarter,
         sum(override_ind=1 and OVRD_SEVERITY>0)  as downgrades,
         sum(override_ind=1 and OVRD_SEVERITY<0)  as upgrades,
         sum(OVRD_SEVERITY> 1) as mat_downgrade,
         sum(OVRD_SEVERITY<-1) as mat_upgrade
  from _ABL_enriched
  group by quarter
  order by quarter;
quit;

/* -- D) Exposure-weighted material impact -- */
proc sql;
  create table abl_mat_amounts as
  select quarter,
         sum(tfc_face_amt)                                             as face_amt_total,
         sum(case when material_1notch=1 then tfc_face_amt else 0 end) as face_amt_material,
         calculated face_amt_material / max(calculated face_amt_total,1) 
           as face_amt_material_share format=percent8.2,
         sum(TFC_Curr_Bal)                                            as curr_bal_total,
         sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end) as curr_bal_material,
         calculated curr_bal_material / max(calculated curr_bal_total,1) 
           as curr_bal_material_share format=percent8.2
  from _ABL_enriched
  group by quarter
  order by quarter;
quit;

/* -- E) Where do (material) overrides concentrate? (heritage / LOB / reason / source system) -- */
proc sql;
  create table abl_mat_by_heritage as
  select quarter, coalesce(legacy_bank_new,'(missing)') as heritage,
         count(*) as obs, sum(material_1notch) as mat_cnt,
         calculated mat_cnt / calculated obs as mat_rate format=percent8.2
  from _ABL_enriched
  group by quarter, calculated heritage
  order by quarter, mat_rate desc;
quit;

proc sql;
  create table abl_mat_by_lob as
  select quarter, coalesce(STI_lob_nm,'(missing)') as lob,
         coalesce(STI_sub_lob_nm,'(missing)') as sub_lob,
         count(*) as obs, sum(material_1notch) as mat_cnt,
         calculated mat_cnt / calculated obs as mat_rate format=percent8.2
  from _ABL_enriched
  group by quarter, calculated lob, calculated sub_lob
  order by quarter, mat_rate desc;
quit;

proc sql;
  create table abl_mat_by_reason as
  select quarter, coalesce(Override_Reason,'(missing)') as override_reason,
         count(*) as obs, sum(override_ind) as ovr_cnt, sum(material_1notch) as mat_cnt,
         calculated mat_cnt / calculated obs as mat_rate format=percent8.2
  from _ABL_enriched
  group by quarter, calculated override_reason
  order by quarter, mat_cnt desc;
quit;

proc sql;
  create table abl_mat_by_srcsys as
  select quarter, coalesce(TFC_src_sys_cd,'(missing)') as src_sys,
         count(*) as obs, sum(material_1notch) as mat_cnt,
         calculated mat_cnt / calculated obs as mat_rate format=percent8.2
  from _ABL_enriched
  group by quarter, calculated src_sys
  order by quarter, mat_rate desc;
quit;

proc sql;
  create table abl_mat_by_ratingmodel as
  select quarter, coalesce(STI_rating_model,'(missing)') as rating_model,
         count(*) as obs, sum(material_1notch) as mat_cnt,
         calculated mat_cnt / calculated obs as mat_rate format=percent8.2
  from _ABL_enriched
  group by quarter, calculated rating_model
  order by quarter, mat_rate desc;
quit;

/* -- F) Repeats: where do they cluster? (join flags to attributes) -- */
proc sql;
  create table _flags_join as
  select f.*, a.legacy_bank_new, a.STI_lob_nm, a.STI_sub_lob_nm, a.Override_Reason
  from _ABL_flags f
  left join _ABL_attrs a
    on f.quarter=a.quarter and f.c_obgobl=a.c_obgobl;
quit;

proc sql;
  create table abl_repeat_by_heritage as
  select quarter, coalesce(legacy_bank_new,'(missing)') as heritage,
         sum(material_1notch) as mat_total,
         sum(repeat_material) as mat_repeat,
         calculated mat_repeat / max(calculated mat_total,1) as mat_repeat_pct format=percent8.2
  from _flags_join
  group by quarter, calculated heritage
  order by quarter, mat_repeat_pct desc;
quit;

proc sql;
  create table abl_repeat_by_lob as
  select quarter, coalesce(STI_lob_nm,'(missing)') as lob,
         sum(material_1notch) as mat_total,
         sum(repeat_material) as mat_repeat,
         calculated mat_repeat / max(calculated mat_total,1) as mat_repeat_pct format=percent8.2
  from _flags_join
  group by quarter, calculated lob
  order by quarter, mat_repeat_pct desc;
quit;

proc sql;
  create table abl_repeat_by_reason as
  select quarter, coalesce(Override_Reason,'(missing)') as reason,
         sum(material_1notch) as mat_total,
         sum(repeat_material) as mat_repeat,
         calculated mat_repeat / max(calculated mat_total,1) as mat_repeat_pct format=percent8.2
  from _flags_join
  group by quarter, calculated reason
  order by quarter, mat_repeat_pct desc;
quit;

/* -- G) Consecutive repeats (immediately prior quarter also material) -- */
data _ABL_flags2;
  set _ABL_flags;
  by obligor_key qtr_idx;
  retain last_mat_qtr_idx .;
  if first.obligor_key then last_mat_qtr_idx=.;
  consec_material = 0;
  if material_1notch=1 then do;
    if not missing(last_mat_qtr_idx) and qtr_idx = last_mat_qtr_idx + 1 then consec_material=1;
    last_mat_qtr_idx = qtr_idx;
  end;
run;

proc sql;
  create table abl_consecutive_share as
  select quarter,
         sum(material_1notch) as mat_total,
         sum(consec_material) as mat_consec,
         calculated mat_consec / max(calculated mat_total,1) as mat_consec_pct format=percent8.2
  from _ABL_flags2
  group by quarter
  order by quarter;
quit;
