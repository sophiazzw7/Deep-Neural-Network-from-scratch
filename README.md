/* ---------- safe cleanup (do NOT purge abl_material_repeat_by_qtr) ---------- */
proc datasets lib=work nolist;
  delete _abl_attrs _abl_enriched _flags_join _reason_seq _reason_stick _t1 _t2 _t3 _t4
         abl_mat_obl_per_obg abl_mat_obl_per_obg_dist;
quit;

/* ---------- build one attributes table with consistent wide lengths ---------- */
data _abl_attrs;
  length quarter $7 c_obgobl $200 c_obg 8 c_obl 8
         legacy_bank_new $100 business_group $100
         STI_lob_nm $400 STI_sub_lob_nm $400
         TFC_src_sys_cd $200 STI_rating_model $200 LASFACILITYNBR $200
         Override_Reason $200
         TFC_Curr_Bal 8 tfc_face_amt 8;
  set
    ogm.OVERRIDE_2024Q2_DEDUP_ABL(in=i1 keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt Override_Reason
                                         legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
                                         TFC_src_sys_cd STI_rating_model LASFACILITYNBR
                                   rename=(Override_Reason=_or))
    ogm.OVERRIDE_2024Q3_DEDUP_ABL(in=i2 keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt Override_Reason
                                         legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
                                         TFC_src_sys_cd STI_rating_model LASFACILITYNBR
                                   rename=(Override_Reason=_or))
    ogm.OVERRIDE_2024Q4_DEDUP_ABL(in=i3 keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt Override_Reason
                                         legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
                                         TFC_src_sys_cd STI_rating_model LASFACILITYNBR
                                   rename=(Override_Reason=_or))
    ogm.OVERRIDE_2025Q1_DEDUP_ABL(in=i4 keep=c_obg c_obl c_obgobl TFC_Curr_Bal tfc_face_amt Override_Reason
                                         legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm
                                         TFC_src_sys_cd STI_rating_model LASFACILITYNBR
                                   rename=(Override_Reason=_or))
  ;
  if i1 then quarter='2024Q2';
  else if i2 then quarter='2024Q3';
  else if i3 then quarter='2024Q4';
  else if i4 then quarter='2025Q1';
  Override_Reason = strip(_or);
  drop _or;
run;

proc sort data=_abl_attrs nodupkey; by quarter c_obgobl; run;

/* ---------- join attributes to your obligation-level table ---------- */
proc sql;
  create table _abl_enriched as
  select a.*,
         b.c_obg, b.c_obl, b.TFC_Curr_Bal, b.tfc_face_amt, b.Override_Reason,
         b.legacy_bank_new, b.business_group, b.STI_lob_nm, b.STI_sub_lob_nm,
         b.TFC_src_sys_cd, b.STI_rating_model, b.LASFACILITYNBR
  from _abl as a
  left join _abl_attrs as b
    on a.quarter=b.quarter and a.c_obgobl=b.c_obgobl;
quit;

/* ---------- inflation & distributions ---------- */
proc sql;
  create table abl_inflation_qtr as
  select quarter,
         sum(material_1notch) as mat_obligations,
         count(distinct case when material_1notch=1 then c_obg end) as mat_obligors,
         calculated mat_obligations - calculated mat_obligors as extra_obligations,
         calculated mat_obligations / max(calculated mat_obligors,1) as obligs_per_obligor format=8.2
  from _abl_enriched
  group by quarter
  order by quarter;
quit;

proc sql;
  create table abl_mat_obl_per_obg as
  select quarter, c_obg, count(*) as mat_obligations
  from _abl_enriched
  where material_1notch=1
  group by quarter, c_obg;
quit;

proc sql;
  create table abl_mat_obl_per_obg_dist as
  select quarter, mat_obligations, count(*) as obligors
  from abl_mat_obl_per_obg
  group by quarter, mat_obligations
  order by quarter, mat_obligations;
quit;

/* ---------- severity & direction ---------- */
data _abl_enriched;
  set _abl_enriched;
  length sev_bucket $6;
  if OVRD_SEVERITY>=3 then sev_bucket='>=+3';
  else if OVRD_SEVERITY=2 then sev_bucket='+2';
  else if OVRD_SEVERITY=1 then sev_bucket='+1';
  else if OVRD_SEVERITY=0 then sev_bucket='0';
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
         calculated mat_cnt/ calculated obs as mat_rate format=percent8.2,
         mean(OVRD_SEVERITY) as avg_sev format=8.2
  from _abl_enriched
  group by quarter, sev_bucket
  order by quarter, sev_bucket;
quit;

proc sql;
  create table abl_direction as
  select quarter,
         sum(override_ind=1 and OVRD_SEVERITY>0)  as downgrades,
         sum(override_ind=1 and OVRD_SEVERITY<0)  as upgrades,
         sum(OVRD_SEVERITY>1)  as mat_downgrades,
         sum(OVRD_SEVERITY<-1) as mat_upgrades
  from _abl_enriched
  group by quarter
  order by quarter;
quit;

/* ---------- exposure share ---------- */
proc sql;
  create table abl_exposure_share as
  select quarter,
         sum(tfc_face_amt) as face_total,
         sum(case when material_1notch=1 then tfc_face_amt else 0 end) as face_material,
         calculated face_material/max(calculated face_total,1) as face_mat_share format=percent8.2,
         sum(TFC_Curr_Bal) as bal_total,
         sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end) as bal_material,
         calculated bal_material/max(calculated bal_total,1) as bal_mat_share format=percent8.2
  from _abl_enriched
  group by quarter
  order by quarter;
quit;

/* ---------- clustering ---------- */
proc sql;
  create table abl_by_heritage as
  select quarter, coalesce(legacy_bank_new,'(missing)') as heritage,
         count(*) as obs, sum(material_1notch) as mat_cnt,
         calculated mat_cnt/calculated obs as mat_rate format=percent8.2
  from _abl_enriched
  group by quarter, calculated heritage
  order by quarter, mat_rate desc;
quit;

proc sql;
  create table abl_by_lob as
  select quarter, coalesce(STI_lob_nm,'(missing)') as lob,
         coalesce(STI_sub_lob_nm,'(missing)') as sub_lob,
         count(*) as obs, sum(material_1notch) as mat_cnt,
         calculated mat_cnt/calculated obs as mat_rate format=percent8.2
  from _abl_enriched
  group by quarter, calculated lob, calculated sub_lob
  order by quarter, mat_rate desc;
quit;

proc sql;
  create table abl_by_reason as
  select quarter, coalesce(Override_Reason,'(missing)') as reason,
         count(*) as obs, sum(override_ind) as ovr_cnt, sum(material_1notch) as mat_cnt,
         calculated mat_cnt/calculated obs as mat_rate format=percent8.2
  from _abl_enriched
  group by quarter, calculated reason
  order by quarter, mat_cnt desc;
quit;

/* ---------- repeat clustering & reason stickiness (with proper sorts) ---------- */
proc sort data=_abl_enriched; by quarter c_obgobl; run;
proc sort data=_abl_flags;     by quarter c_obgobl; run;

data _reason_seq;
  merge _abl_enriched(in=a keep=quarter c_obgobl c_obg material_1notch Override_Reason qtr_idx)
        _abl_flags   (in=b keep=quarter c_obgobl qtr_idx material_1notch repeat_material);
  by quarter c_obgobl;
  if a;
run;

proc sort data=_reason_seq; by c_obg qtr_idx; run;

data _reason_stick;
  set _reason_seq;
  by c_obg qtr_idx;
  retain prev_reason '';
  if first.c_obg then prev_reason='';
  same_reason_repeat = 0;
  if material_1notch=1 then do;
    if repeat_material=1 and not missing(prev_reason) and upcase(strip(Override_Reason))=upcase(strip(prev_reason)) then same_reason_repeat=1;
    prev_reason=Override_Reason;
  end;
run;

proc sql;
  create table abl_reason_stickiness as
  select quarter,
         sum(material_1notch) as mat_total,
         sum(repeat_material) as mat_repeat,
         sum(same_reason_repeat) as same_reason,
         calculated same_reason/max(calculated mat_repeat,1) as same_reason_pct format=percent8.2
  from _reason_stick
  group by quarter
  order by quarter;
quit;

/* ---------- concise prints ---------- */
title "Material Repeat by Quarter";
proc print data=abl_material_repeat_by_qtr noobs label; 
  var quarter total_observations mat_total mat_repeat mat_repeat_pct mat_first_time mat_first_time_pct;
run;

title "Inflation (Obligations vs Unique Obligors)";
proc print data=abl_inflation_qtr noobs label; 
  var quarter mat_obligations mat_obligors extra_obligations obligs_per_obligor;
run;

title "Multi-Obligation Among Material (Top Dist)";
proc print data=abl_mat_obl_per_obg_dist noobs; 
  where mat_obligations>=2;
  var quarter mat_obligations obligors;
run;

title "Exposure Share of Material";
proc print data=abl_exposure_share noobs; 
  var quarter face_total face_material face_mat_share bal_total bal_material bal_mat_share;
run;

title "Direction";
proc print data=abl_direction noobs; 
  var quarter downgrades upgrades mat_downgrades mat_upgrades;
run;

proc sort data=abl_by_heritage out=_t1; by quarter descending mat_rate; run;
title "Where Materials Concentrate (Heritage, top 10 by rate each quarter)";
proc print data=_t1(obs=10) noobs; var quarter heritage obs mat_cnt mat_rate; run;

proc sort data=abl_by_lob out=_t2; by quarter descending mat_rate; run;
title "Where Materials Concentrate (LOB, top 10)";
proc print data=_t2(obs=10) noobs; var quarter lob sub_lob obs mat_cnt mat_rate; run;

proc sort data=abl_by_reason out=_t3; by quarter descending mat_cnt; run;
title "Reasons (top 10 by material count)";
proc print data=_t3(obs=10) noobs; var quarter reason obs ovr_cnt mat_cnt mat_rate; run;

title "Same-Reason Repeats";
proc print data=abl_reason_stickiness noobs; 
  var quarter mat_total mat_repeat same_reason same_reason_pct;
run;
title;
