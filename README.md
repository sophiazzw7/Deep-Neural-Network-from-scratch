/* ======================= SETUP ======================= */
options mprint mlogic symbolgen nosource2;
libname ogm "/sasdata/mrmg1";

/* ---- INPUT TABLES (edit names if yours differ) ---- */
%let DS_ALL   = ogm.all_table_2503_1;           /* OGM instrument snapshot */
%let DS_F_ABL = ogm.abl_lgd_factor_2503_abl;    /* Factor output (has f_fac/productid) */
%let DS_ACP   = ogm.asset_collateral_product;   /* Asset_Collateral_product */

/* =================== 1) BASE SNAPSHOT =================== */
/* Keep only the fields Kevin needs and compute utilization */
data BASE0;
  set &DS_ALL;
  length segment $3 risk_grade $6 lgd_grade $1 facility_type $40;
  segment      = ifc(SF=1,'SF','ABL');
  facility_type= coalescec(strip(STI_lob_nm),'');
  risk_grade   = strip(TFC_rsk_grd);
  lgd_grade    = strip(final_lgd_grade);
  commit       = tfc_face_amt;
  drawn        = tfc_curr_bal;
  asof_dt      = datepart(f_uplddt);
  format asof_dt date9.;
  if commit>0 then util = min(max(drawn/commit,0),1);
  keep segment facility_type risk_grade lgd_grade commit drawn util
       collateralclass asof_dt
       c_obg c_obl TFC_Account_Nbr TFC_Account_Nbr_New;
run;

/* =================== 2) COLLATERAL ENRICHMENT =================== */
/* 2a. Build c_obg/c_obl -> productid (f_fac) map from factor output */
proc contents data=&DS_F_ABL noprint out=_ff_vars_; run;

proc sql noprint;
  /* 1 if f_fac is character */
  select (type=2) into :is_char_ffac
  from _ff_vars_
  where upcase(name)='F_FAC';
quit;

%macro make_fac_key;
  %if &is_char_ffac=1 %then %do;
    proc sql;
      create table FAC_KEY as
      select distinct c_obg, c_obl,
             input(strip(f_fac), best32.) as productid
      from &DS_F_ABL
      where not missing(c_obg) and not missing(c_obl) and not missing(f_fac);
    quit;
  %end;
  %else %do;
    proc sql;
      create table FAC_KEY as
      select distinct c_obg, c_obl, f_fac as productid
      from &DS_F_ABL
      where not missing(c_obg) and not missing(c_obl) and not missing(f_fac);
    quit;
  %end;
%mend;
%make_fac_key;

/* 2b. Attach productid to BASE0 (ABL rows will populate productid; SF typically stays missing) */
proc sql;
  create table BASE1 as
  select a.*, b.productid
  from BASE0 as a
  left join FAC_KEY as b
    on a.c_obg=b.c_obg and a.c_obl=b.c_obl;
quit;

/* 2c. Join to Asset_Collateral_product with dt <= asof_dt, then pick the latest dt per row */
proc sql;
  create table ACP_JOIN as
  select a.*,
         acp.collateralclass as acp_collateralclass,
         acp.dt              as acp_dt
  from BASE1 as a
  left join &DS_ACP as acp
    on a.productid = acp.productid
   and acp.dt     <= a.asof_dt;
quit;

proc sort data=ACP_JOIN;
  by TFC_Account_Nbr c_obg c_obl asof_dt productid acp_dt;
run;

data BASE2;
  set ACP_JOIN;
  by TFC_Account_Nbr c_obg c_obl asof_dt productid acp_dt;
  /* keep the latest eligible ACP record per facility row */
  if first.productid then _keep=.;
  if last.productid then do;
    /* fill collateral if missing in all_table */
    collateralclass_fix = coalesce(collateralclass, acp_collateralclass);
    output;
  end;
  drop _keep acp_collateralclass acp_dt;
run;

/* Final snapshot for Kevin */
data KEVIN_SNAPSHOT;
  set BASE2;
  collateralclass = collateralclass_fix;
  drop collateralclass_fix;
run;

/* =================== 3) QUICK CHECKS =================== */
proc sql;
  create table QC as
  select segment,
         count(*)                           as n_rows,
         sum(commit>0)                      as n_with_commit,
         mean(util)                         as avg_util format=percent8.2,
         mean(missing(collateralclass))     as pct_missing_collat format=percent8.2
  from KEVIN_SNAPSHOT
  group by 1;
quit;

/* =================== 4) UTILIZATION BY RISK =================== */
/* EAD-weighted (sum drawn / sum commit). Suppress sparse bins (n<10). */
proc sql;
  create table UTIL_BY_RISK as
  select risk_grade,
         count(*)                 as n_facilities,
         sum(drawn) / sum(commit) as util_ead_wtd format=percent8.2
  from KEVIN_SNAPSHOT
  where commit>0
  group by 1
  having n_facilities >= 10
  order by risk_grade;
quit;

/* =================== 5) UTILIZATION BY RISK x COLLATERAL (ABL only) =================== */
proc sql;
  create table UTIL_BY_RISK_COLLAT as
  select risk_grade,
         collateralclass,
         count(*)                 as n_facilities,
         sum(drawn) / sum(commit) as util_ead_wtd format=percent8.2
  from KEVIN_SNAPSHOT
  where segment='ABL' and commit>0 and not missing(collateralclass)
  group by 1,2
  having n_facilities >= 10
  order by risk_grade, collateralclass;
quit;

/* =================== 6) OPTIONAL: MIGRATION VIEW (time series, if asof varies) =================== */
proc sql;
  create table UTIL_TS as
  select intnx('qtr', asof_dt, 0, 'b') as asof_qtr format=yyq6.,
         risk_grade,
         count(*)                 as n_facilities,
         sum(drawn) / sum(commit) as util_ead_wtd format=percent8.2
  from KEVIN_SNAPSHOT
  where commit>0 and not missing(asof_dt)
  group by 1,2
  having n_facilities >= 10
  order by asof_qtr, risk_grade;
quit;

/* =================== 7) OPTIONAL: EXPORT TO EXCEL =================== */
/* Uncomment and set a path if you want an .xlsx extract
filename xout "/sasdata/mrmg1/kevin_utilization_results.xlsx";
proc export data=KEVIN_SNAPSHOT outfile=xout dbms=xlsx replace; sheet="Snapshot"; run;
proc export data=UTIL_BY_RISK   outfile=xout dbms=xlsx replace; sheet="Util_by_Risk"; run;
proc export data=UTIL_BY_RISK_COLLAT outfile=xout dbms=xlsx replace; sheet="Util_by_Risk_Collat"; run;
proc export data=UTIL_TS        outfile=xout dbms=xlsx replace; sheet="Util_TS"; run;
filename xout clear;
*/
