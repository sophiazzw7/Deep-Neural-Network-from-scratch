/*============ 0) DATA LOCATIONS (edit these) ============*/
libname ogm "/sasdata/mrmg1";

%let DS_ALL   = ogm.all_table_2503_1;          /* instrument table (OGM export) */
%let DS_F_ABL = ogm.abl_lgd_factor_2503_abl;   /* ABL factor output (has f_fac) */
%let DS_ACP   = ogm.asset_collateral_product;  /* Asset_Collateral_product */

/*============ 1) BASE SNAPSHOT FROM ALL_TABLE ============*/
data BASE0;
  set &DS_ALL;
  length segment $3;
  segment   = ifc(SF=1,'SF','ABL');
  commit    = tfc_face_amt;
  drawn     = tfc_curr_bal;
  risk_grade= strip(TFC_rsk_grd);
  lgd_grade = strip(final_lgd_grade);
  asof_dt   = datepart(f_uplddt);
  format asof_dt date9.;
  if commit>0 then util = min(max(drawn/commit,0),1);
  keep segment STI_lob_nm commit drawn util risk_grade lgd_grade
       collateralclass asof_dt TFC_Account_Nbr TFC_Account_Nbr_New
       c_obg c_obl;
run;

/*============ 2) COLLATERAL ENRICHMENT (only when missing) ============*/
/* 2a. factor → productid key for ABL (c_obg/c_obl → f_fac) */
proc sql;
  create table FAC_KEY as
  select distinct c_obg, c_obl, f_fac as productid
  from &DS_F_ABL
  where not missing(c_obg) and not missing(c_obl) and not missing(f_fac);
quit;

/* 2b. latest collateralclass per productid from Asset_Collateral_product */
proc sql;
  create table ACP_LATEST as
  select productid,
         collateralclass,
         /* pick latest dt per productid */
         max(dt) as dt_keep
  from &DS_ACP
  group by productid;
quit;

proc sql;
  create table COLLAT_MAP as
  select a.productid, b.collateralclass
  from ACP_LATEST a
  left join &DS_ACP b
    on a.productid=b.productid and a.dt_keep=b.dt;
quit;

/* 2c. fill missing collateralclass in the base using factor→ACP mapping */
proc sql;
  create table BASE as
  select b0.*,
         coalesce(b0.collateralclass, cm.collateralclass) as collateralclass_fix
  from BASE0 b0
  left join FAC_KEY fk
    on b0.c_obg=fk.c_obg and b0.c_obl=fk.c_obl
  left join COLLAT_MAP cm
    on fk.productid=cm.productid;
quit;

data KEVIN_SNAPSHOT;
  set BASE;
  collateralclass = collateralclass_fix;
  length collateral_label $12;
  collateral_label = ifc(collateralclass=4,'AR',
                    ifc(collateralclass=5,'AR-Gov',
                    ifc(collateralclass=6,'Inventory','')));
  facility_type = strip(STI_lob_nm);
  drop collateralclass_fix STI_lob_nm c_obg c_obl;
run;

/*============ 3) QUICK QC ============*/
proc sql;
  create table QC as
  select segment,
         count(*)                             as n_facilities,
         sum(commit>0)                        as n_with_commit,
         sum(drawn>commit and commit>0)       as n_overdrawn,
         mean(missing(collateralclass))       as pct_missing_collat format=percent8.2,
         mean(util)                           as avg_util_raw format=percent8.2
  from KEVIN_SNAPSHOT
  group by 1;
quit;

/*============ 4) PIVOT: UTILIZATION BY RISK (EAD-WEIGHTED) ============*/
proc sql;
  create table UTIL_BY_RISK as
  select risk_grade,
         count(*)                             as n_facilities,
         sum(drawn) / sum(commit)             as util_ead_wtd format=percent8.2
  from KEVIN_SNAPSHOT
  where commit>0
  group by 1
  having n_facilities>=10
  order by risk_grade;
quit;

/*============ 5) PIVOT: UTILIZATION BY RISK × COLLATERAL (ABL only) ============*/
proc sql;
  create table UTIL_BY_RISK_COLLAT as
  select risk_grade,
         collateralclass,
         count(*)                             as n_facilities,
         sum(drawn) / sum(commit)             as util_ead_wtd format=percent8.2
  from KEVIN_SNAPSHOT
  where segment='ABL' and commit>0 and collateralclass in (4,5,6)
  group by 1,2
  having n_facilities>=10
  order by risk_grade, collateralclass;
quit;

/*============ 6) OPTIONAL: TIME-SERIES FOR MIGRATION VIEW ============*/
proc sql;
  create table UTIL_TS as
  select intnx('qtr',asof_dt,0,'b') as asof_qtr format=yyq6.,
         risk_grade,
         sum(drawn) / sum(commit)   as util_ead_wtd format=percent8.2,
         count(*) as n_facilities
  from KEVIN_SNAPSHOT
  where commit>0 and not missing(asof_dt)
  group by 1,2
  having n_facilities>=10
  order by 1,2;
quit;
