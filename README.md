/* ----- Join flags to attributes (exposure fields) and build utilization ----- */
proc sql;
  create table _flags_expo as
  select f.quarter, f.c_obgobl, f.obligor_key, f.material_1notch, f.override_ind,
         f.repeat_material, f.qtr_idx, f.OVRD_SEVERITY,
         e.c_obg, e.tfc_face_amt, e.TFC_Curr_Bal, e.Override_Reason,
         e.legacy_bank_new, e.STI_lob_nm, e.STI_sub_lob_nm
  from _abl_flags f
  left join _abl_enriched e
    on f.quarter=e.quarter and f.c_obgobl=e.c_obgobl;
quit;

/* Basic QC on exposure fields */
proc sql;
  create table util_qc as
  select quarter,
         sum(missing(tfc_face_amt)) as miss_face,
         sum(missing(TFC_Curr_Bal)) as miss_bal,
         sum(tfc_face_amt<=0)       as nonpos_face,
         sum(TFC_Curr_Bal<0)        as neg_bal
  from _flags_expo
  group by quarter
  order by quarter;
quit;

/* Utilization with sensible caps; drop rows with non-positive face */
data _util_base;
  set _flags_expo;
  if tfc_face_amt>0 then do;
    util = TFC_Curr_Bal / tfc_face_amt;
    if util < 0 then util = 0;
    if util > 2 then util = 2;  /* cap at 200% to tame outliers */
  end;
  else delete;

  length util_bucket $10;
  if      util < 0.25 then util_bucket='<25%';
  else if util < 0.50 then util_bucket='25–50%';
  else if util < 0.75 then util_bucket='50–75%';
  else if util < 0.90 then util_bucket='75–90%';
  else if util <=1.00 then util_bucket='90–100%';
  else                      util_bucket='>100%';
run;

/* Exposure-weighted material share by quarter & utilization bucket */
proc sql;
  create table util_bucket_qtr as
  select quarter, util_bucket,
         count(*)                                        as obs,
         sum(override_ind)                               as ovr_cnt,
         sum(material_1notch)                            as mat_cnt,
         calculated ovr_cnt / calculated obs             as ovr_rate format=percent8.2,
         calculated mat_cnt / calculated obs             as mat_rate format=percent8.2,
         sum(TFC_Curr_Bal)                               as bal_total,
         sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end) as bal_mat,
         calculated bal_mat / max(calculated bal_total,1) as bal_mat_share format=percent8.2
  from _util_base
  group by quarter, util_bucket
  order by quarter, util_bucket;
quit;

/* Deciles of utilization within each quarter; severity & rates by decile */
proc rank data=_util_base groups=10 out=_util_dec; by quarter; var util; ranks util_decile; run;

proc sql;
  create table util_decile_qtr as
  select quarter, util_decile+1 as decile,
         count(*) as obs,
         mean(util) as avg_util format=percent8.2,
         mean(OVRD_SEVERITY) as avg_sev format=8.2,
         sum(material_1notch)/count(*) as mat_rate format=percent8.2
  from _util_dec
  group by quarter, util_decile
  order by quarter, decile;
quit;

/* Repeats by utilization: of this quarter’s MATERIAL cases, what share are repeats in each bucket */
proc sql;
  create table repeat_by_util_qtr as
  select quarter, util_bucket,
         sum(material_1notch)               as mat_total,
         sum(case when material_1notch=1 and repeat_material=1 then 1 else 0 end) as mat_repeat,
         calculated mat_repeat / max(calculated mat_total,1) as repeat_pct format=percent8.2
  from _util_base
  group by quarter, util_bucket
  order by quarter, util_bucket;
quit;

/* Reason mix by utilization for top reasons (by total material count) */
proc sql;
  create table _top_reasons as
  select Override_Reason as reason,
         sum(material_1notch) as mat_cnt
  from _util_base
  group by Override_Reason
  having calculated mat_cnt>0
  order by mat_cnt desc;
quit;

proc sql outobs=5;
  create table top5_reasons as
  select * from _top_reasons;
quit;

proc sql;
  create table reason_by_util_qtr as
  select u.quarter, u.util_bucket, 
         coalesce(u.Override_Reason,'(missing)') as reason,
         count(*) as obs, sum(material_1notch) as mat_cnt,
         calculated mat_cnt / calculated obs as mat_rate format=percent8.2
  from _util_base u
  where exists (select 1 from top5_reasons t where t.reason = u.Override_Reason)
  group by u.quarter, u.util_bucket, calculated reason
  order by u.quarter, calculated reason, u.util_bucket;
quit;

/* Simple predictive check: does higher utilization predict material overrides? */
proc logistic data=_util_base noprint outest=logit_util_estimates;
  class quarter / param=ref ref='2024Q2';
  model material_1notch(event='1') = util quarter;
run;

/* Prints */
title "Utilization QC (missing/non-positive exposure fields)";
proc print data=util_qc noobs; run;

title "Material/Override Rates by Utilization Bucket (incl. balance-weighted share)";
proc print data=util_bucket_qtr noobs label;
  var quarter util_bucket obs ovr_cnt ovr_rate mat_cnt mat_rate bal_mat_share;
  label bal_mat_share='bal-weighted mat%';
run;

title "Utilization Deciles – severity and material rate";
proc print data=util_decile_qtr noobs; 
  var quarter decile avg_util avg_sev mat_rate;
run;

title "Material Repeats by Utilization Bucket";
proc print data=repeat_by_util_qtr noobs;
  var quarter util_bucket mat_total mat_repeat repeat_pct;
run;

title "Top Reasons by Utilization Bucket (Top 5 reasons overall)";
proc print data=reason_by_util_qtr noobs;
  var quarter reason util_bucket obs mat_cnt mat_rate;
run;

title "Logistic Check: util coefficient by quarter (positive coef => higher util ↑ prob of material)";
proc print data=logit_util_estimates noobs;
  var _LNLIKE_ Intercept util;
run;
title;
