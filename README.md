/* ---------- 0) Point to your all_table ---------- */
libname ogm "/sasdata/mrmg1";             /* <-- change if needed */
%let DS_IN = ogm.all_table_2503_1;        /* <-- your all_table dataset */

/* ---------- 1) Kevin snapshot (keep only essentials) ---------- */
data KEVIN_SNAPSHOT;
  set &DS_IN;

  /* segment */
  length segment $3;
  segment = ifc(SF=1,'SF','ABL');

  /* core fields */
  length facility_type $40 collateral_label $20;
  facility_type   = strip(STI_lob_nm);                /* instrument-level type */
  commitment_amt  = tfc_face_amt;
  drawn_amt       = tfc_curr_bal;
  risk_grade      = strip(TFC_rsk_grd);               /* borrower risk rating */
  lgd_grade       = strip(final_lgd_grade);           /* LGD rating letter */
  collateralclass = collateralclass;                  /* 4=AR,5=AR-Gov,6=Inv (ABL only) */
  collateral_label = catx('',
         ifc(collateralclass=4,'AR',
         ifc(collateralclass=5,'AR-Gov',
         ifc(collateralclass=6,'Inventory',''))),'');
  asof_dt         = datepart(f_uplddt);               /* age/vintage (upload date) */
  format asof_dt date9.;

  /* utilization */
  if commitment_amt>0 then util = max(0,min(1, drawn_amt/commitment_amt));
  else util = .;

  keep segment facility_type commitment_amt drawn_amt util
       risk_grade lgd_grade collateralclass collateral_label asof_dt
       TFC_Account_Nbr TFC_Account_Nbr_New; /* ids to keep for traceability */
run;

/* ---------- 2) Quick hygiene checks (one line each) ---------- */
proc sql;
  /* coverage and sanity */
  create table QC as
  select
    segment,
    count(*)                                  as n_facilities,
    sum(commitment_amt>0)                     as n_with_commit,
    mean(util)                                as avg_util_raw format=percent8.2,
    sum(drawn_amt>commitment_amt and commitment_amt>0) as n_overdrawn,
    mean(missing(collateralclass))            as pct_missing_collat format=percent8.2
  from KEVIN_SNAPSHOT
  group by 1;

  /* violations detail (optional) */
  create table QC_OVER as
  select *
  from KEVIN_SNAPSHOT
  where commitment_amt>0 and drawn_amt>commitment_amt;
quit;

/* ---------- 3) Pivot: utilization by risk grade (EAD-weighted) ---------- */
proc sql;
  create table UTIL_BY_RISK as
  select
    risk_grade,
    count(*)                                as n_facilities,
    sum(drawn_amt) / sum(commitment_amt)    as util_ead_wtd format=percent8.2
  from KEVIN_SNAPSHOT
  where commitment_amt>0
  group by 1
  having n_facilities>=10                   /* hide thin bins; tweak if needed */
  order by calculated util_ead_wtd;
quit;

/* ---------- 4) Pivot: utilization by risk × collateral (ABL only) ---------- */
proc sql;
  create table UTIL_BY_RISK_COLLAT as
  select
    risk_grade,
    collateralclass,
    count(*)                                as n_facilities,
    sum(drawn_amt) / sum(commitment_amt)    as util_ead_wtd format=percent8.2
  from KEVIN_SNAPSHOT
  where segment='ABL' and commitment_amt>0 and not missing(collateralclass)
  group by 1,2
  having n_facilities>=10
  order by risk_grade, collateralclass;
quit;

/* ---------- 5) (Optional) Vintage view: utilization by quarter & risk ---------- */
proc sql;
  create table UTIL_TS as
  select intnx('qtr',asof_dt,0,'b') as asof_qtr format=yyq6.,
         risk_grade,
         sum(drawn_amt)/sum(commitment_amt) as util_ead_wtd format=percent8.2,
         count(*) as n_facilities
  from KEVIN_SNAPSHOT
  where commitment_amt>0 and not missing(asof_dt)
  group by 1,2
  having n_facilities>=10
  order by 1,2;
quit;
