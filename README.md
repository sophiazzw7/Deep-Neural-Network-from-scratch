/* ========= EDIT: folder & member names ========= */
%let PATH = /sasdata/mrmg1;

%let DS_ALL   = all_table_2503_1;          /* instrument table */
%let DS_F_ABL = abl_lgd_factor_2503_abl;   /* factor output ABL */
%let DS_F_SF  = abl_lgd_factor_2503_sf;    /* factor output SF  */
/* ============================================== */

options mprint mlogic symbolgen;
libname ogm "&PATH";

/* 1) Base instrument snapshot (only what Kevin asked for) */
data base_snapshot;
  set ogm.&DS_ALL;
  keep c_obg c_obl TFC_Account_Nbr customerid
       STI_lob_nm STI_sub_lob_nm business_group
       tfc_face_amt tfc_curr_bal
       tfc_rsk_grd
       model_lgd_grade final_lgd_grade final_lgd_perc
       f_uplddt;
run;

/* 2) Collateral from LGD factor outputs (ABL + SF) */
data factor_coll(keep=c_obg c_obl collateralclass controltypr valuationmethod
                       SUNTRUSTALLOCATIONPERCENT dt);
  set ogm.&DS_F_ABL ogm.&DS_F_SF;
run;

/* Choose dominant collateral per instrument: highest allocation %, then most recent dt */
proc sort data=factor_coll;
  by c_obg c_obl descending SUNTRUSTALLOCATIONPERCENT descending dt;
run;

data coll_one_per_obgobl;
  set factor_coll;
  by c_obg c_obl;
  if first.c_obl then output;
run;

/* 3) Join back to instruments using c_obg + c_obl */
proc sql;
  create table kevin_snapshot as
  select a.*,
         b.collateralclass,
         b.controltypr,
         b.valuationmethod,
         b.SUNTRUSTALLOCATIONPERCENT as coll_alloc_pct
  from base_snapshot as a
  left join coll_one_per_obgobl as b
    on a.c_obg=b.c_obg and a.c_obl=b.c_obl;
quit;

/* 4) Utilization indicator */
data kevin_snapshot;
  set kevin_snapshot;
  if tfc_face_amt>0 then utilization = tfc_curr_bal / tfc_face_amt;
  else utilization = .;
run;

/* 5) Pivots requested */
proc sql;
  /* by borrower risk rating */
  create table util_by_risk as
  select tfc_rsk_grd,
         count(*)      as n_facilities,
         mean(utilization) as avg_utilization
  from kevin_snapshot
  group by tfc_rsk_grd
  order by tfc_rsk_grd;

  /* by borrower risk rating x collateral class */
  create table util_by_risk_collat as
  select tfc_rsk_grd, collateralclass,
         count(*) as n_facilities,
         mean(utilization) as avg_utilization
  from kevin_snapshot
  group by tfc_rsk_grd, collateralclass
  order by tfc_rsk_grd, collateralclass;
quit;
