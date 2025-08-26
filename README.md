/* ================= EDIT THESE ================= */
%let PATH         = /sasdata/mrmg1;            /* folder with all datasets */
%let DS_ALL_CURR  = all_table_2503_1;          /* current quarter all_table */
%let DS_F_ABL     = abl_lgd_factor_2503_abl;   /* factor output ABL (current) */
%let DS_F_SF      = abl_lgd_factor_2503_sf;    /* factor output SF  (current) */

/* Optional time-series/migration (leave blank if not available) */
%let DS_ALL_PREV  = ;                           /* e.g., all_table_2403_1     */
%let DS_F_ABL_PREV= ;                           /* e.g., abl_lgd_factor_2403_abl */
%let DS_F_SF_PREV = ;                           /* e.g., abl_lgd_factor_2403_sf  */

/* Choose collateral treatment: DOMINANT or ALLOC */
%let COLLATERAL_MODE = DOMINANT;

/* Optional export path (leave blank to skip CSVs) */
%let OUT_PATH     = /sasdata/mrmg1/out;
/* ============================================== */

options mprint mlogic symbolgen;
libname ogm "&PATH";

/* ------------------ Helpers ------------------- */
/* Ensure variable exists in a dataset */
%macro hasvar(ds, var);
  %local dsid rc idx;
  %let dsid = %sysfunc(open(&ds.));
  %if &dsid %then %do;
    %let idx = %sysfunc(varnum(&dsid., &var.));
    %let rc  = %sysfunc(close(&dsid.));
    %sysfunc(ifc(&idx>0,1,0))
  %end;
  %else 0
%mend;

/* Format risk grade as ordered if numeric; otherwise keep character order */
proc format;
  /* Edit mapping if your grades are alpha (e.g., 'Pass','Watch', etc.) */
  value rskord
    low-high = [best12.]; /* fallback numeric */
run;

/* -------------- Build current snapshot -------------- */
/* 1) Instrument spine (only fields Kevin needs) */
data base_snapshot_cur;
  set ogm.&DS_ALL_CURR;
  keep c_obg c_obl TFC_Account_Nbr customerid
       STI_lob_nm STI_sub_lob_nm business_group
       tfc_face_amt tfc_curr_bal
       tfc_rsk_grd
       model_lgd_grade final_lgd_grade final_lgd_perc
       f_uplddt;
run;

/* 2) Collateral from LGD factor outputs (carry c_obg,c_obl + collateral fields) */
data factor_coll_cur(keep=c_obg c_obl collateralclass controltypr valuationmethod
                           SUNTRUSTALLOCATIONPERCENT dt);
  set ogm.&DS_F_ABL ogm.&DS_F_SF;
run;

/* 2a) Dominant collateral per instrument (highest alloc%, then latest dt) */
proc sort data=factor_coll_cur;
  by c_obg c_obl descending SUNTRUSTALLOCATIONPERCENT descending dt;
run;

data coll_dom_cur;
  set factor_coll_cur;
  by c_obg c_obl;
  if first.c_obl then output;
run;

/* 3) Join collateral back to instrument spine */
%macro join_collateral(mode=DOMINANT);
  %if %upcase(&mode.)=DOMINANT %then %do;
    proc sql;
      create table kevin_snapshot_cur as
      select a.*,
             b.collateralclass,
             b.controltypr,
             b.valuationmethod,
             b.SUNTRUSTALLOCATIONPERCENT as coll_alloc_pct
      from base_snapshot_cur as a
      left join coll_dom_cur as b
        on a.c_obg=b.c_obg and a.c_obl=b.c_obl;
    quit;
  %end;
  %else %if %upcase(&mode.)=ALLOC %then %do;
    /* Allocation-aware slices: replicate rows by collateral slice with alloc% */
    proc sql;
      create table kevin_snapshot_cur as
      select a.*,
             b.collateralclass,
             b.controltypr,
             b.valuationmethod,
             b.SUNTRUSTALLOCATIONPERCENT as coll_alloc_pct
      from base_snapshot_cur as a
      left join factor_coll_cur as b
        on a.c_obg=b.c_obg and a.c_obl=b.c_obl;
    quit;
  %end;
  %else %do;
    %put ERROR: COLLATERAL_MODE=&mode not recognized. Use DOMINANT or ALLOC.;
    %abort cancel;
  %end;
%mend;
%join_collateral(mode=&COLLATERAL_MODE.);

/* 4) Utilization; guard div/0 and cap extreme ratios for QA flags (optional) */
data kevin_snapshot_cur;
  set kevin_snapshot_cur;
  if tfc_face_amt>0 then utilization = tfc_curr_bal / tfc_face_amt;
  else utilization = .;
  /* Optional QA: flag extreme utilization >120% */
  length util_flag $10;
  if utilization ne . and utilization>1.2 then util_flag='>120%';
run;

/* -------------- Pivots (current quarter) -------------- */
/* Preferred aggregation = exposure-weighted:
   weighted_avg_util = sum(curr_bal) / sum(face_amt)
   Also compute unweighted mean for reference. */

/* 1) Utilization by Borrower Risk Rating */
proc sql;
  create table pivot_util_by_risk as
  select tfc_rsk_grd,
         count(*)                              as n_facilities,
         sum(coalesce(tfc_face_amt,0))         as total_commitment,
         sum(coalesce(tfc_curr_bal,0))         as total_drawn,
         calculated total_drawn / nullif(calculated total_commitment,0) as util_exp_weighted,
         mean(utilization)                     as util_unweighted
  from kevin_snapshot_cur
  where tfc_face_amt>0
  group by tfc_rsk_grd
  order by tfc_rsk_grd;
quit;

/* 2) Utilization by Risk Rating x Collateral Class */
%macro pivot_util_by_risk_collat;
  %if %upcase(&COLLATERAL_MODE.)=DOMINANT %then %do;
    proc sql;
      create table pivot_util_by_risk_collat as
      select tfc_rsk_grd,
             coalesce(collateralclass,'(Missing)') as collateralclass,
             count(*)                          as n_facilities,
             sum(coalesce(tfc_face_amt,0))     as total_commitment,
             sum(coalesce(tfc_curr_bal,0))     as total_drawn,
             calculated total_drawn / nullif(calculated total_commitment,0) as util_exp_weighted,
             mean(utilization)                 as util_unweighted
      from kevin_snapshot_cur
      where tfc_face_amt>0
      group by tfc_rsk_grd, calculated collateralclass
      order by tfc_rsk_grd, calculated collateralclass;
    quit;
  %end;
  %else %do; /* ALLOC: weight exposures by allocation percent */
    data ks_alloc_cur;
      set kevin_snapshot_cur;
      /* Normalize allocation percent to 0-1 if needed */
      if coll_alloc_pct>. and coll_alloc_pct>1 then coll_w = coll_alloc_pct/100;
      else coll_w = coll_alloc_pct;
      if missing(coll_w) then coll_w=1; /* if missing, treat as 100% */
      face_alloc = coll_w * coalesce(tfc_face_amt,0);
      curr_alloc = coll_w * coalesce(tfc_curr_bal,0);
    run;

    proc sql;
      create table pivot_util_by_risk_collat as
      select tfc_rsk_grd,
             coalesce(collateralclass,'(Missing)') as collateralclass,
             count(*)                          as n_rows,    /* slice rows */
             sum(face_alloc)                   as total_commitment_alloc,
             sum(curr_alloc)                   as total_drawn_alloc,
             calculated total_drawn_alloc / nullif(calculated total_commitment_alloc,0) as util_exp_weighted
      from ks_alloc_cur
      where tfc_face_amt>0
      group by tfc_rsk_grd, calculated collateralclass
      order by tfc_rsk_grd, calculated collateralclass;
    quit;
  %end;
%mend;
%pivot_util_by_risk_collat;

/* 3) Utilization by Facility Type (Sub-LOB) x Risk Rating */
proc sql;
  create table pivot_util_by_fac_risk as
  select STI_sub_lob_nm,
         tfc_rsk_grd,
         count(*)                          as n_facilities,
         sum(coalesce(tfc_face_amt,0))     as total_commitment,
         sum(coalesce(tfc_curr_bal,0))     as total_drawn,
         calculated total_drawn / nullif(calculated total_commitment,0) as util_exp_weighted
  from kevin_snapshot_cur
  where tfc_face_amt>0
  group by STI_sub_lob_nm, tfc_rsk_grd
  order by STI_sub_lob_nm, tfc_rsk_grd;
quit;

/* 4) LGD Grade overlay: Utilization by final LGD grade x risk rating */
proc sql;
  create table pivot_util_lgd_overlay as
  select final_lgd_grade,
         tfc_rsk_grd,
         count(*)                          as n_facilities,
         sum(coalesce(tfc_face_amt,0))     as total_commitment,
         sum(coalesce(tfc_curr_bal,0))     as total_drawn,
         calculated total_drawn / nullif(calculated total_commitment,0) as util_exp_weighted
  from kevin_snapshot_cur
  where tfc_face_amt>0
  group by final_lgd_grade, tfc_rsk_grd
  order by final_lgd_grade, tfc_rsk_grd;
quit;

/* 5) Vintage slice (by snapshot quarter) */
data ks_vintage;
  set kevin_snapshot_cur;
  /* Derive Year-Quarter from f_uplddt; adjust if f_uplddt is char */
  format f_uplddt yymmdd10. vintage_yrqtr $7.;
  if not missing(f_uplddt) then vintage_yrqtr = cats(put(year(f_uplddt),4.), 'Q', put(qtr(f_uplddt),1.));
run;

proc sql;
  create table pivot_util_by_vintage as
  select vintage_yrqtr,
         tfc_rsk_grd,
         count(*)                          as n_facilities,
         sum(coalesce(tfc_face_amt,0))     as total_commitment,
         sum(coalesce(tfc_curr_bal,0))     as total_drawn,
         calculated total_drawn / nullif(calculated total_commitment,0) as util_exp_weighted
  from ks_vintage
  where tfc_face_amt>0
  group by vintage_yrqtr, tfc_rsk_grd
  order by vintage_yrqtr, tfc_rsk_grd;
quit;

/* -------------- Optional: Time-series & Migration -------------- */
%macro build_prev_if_available;
  %if %length(&DS_ALL_PREV.) %then %do;
    /* Prev-quarter instrument */
    data base_snapshot_prev;
      set ogm.&DS_ALL_PREV;
      keep c_obg c_obl TFC_Account_Nbr
           tfc_face_amt tfc_curr_bal tfc_rsk_grd f_uplddt;
    run;

    /* Prev collateral from factors (dominant slice for key only if desired) */
    %if %length(&DS_F_ABL_PREV.) and %length(&DS_F_SF_PREV.) %then %do;
      data factor_coll_prev(keep=c_obg c_obl); /* keys only for join coverage */
        set ogm.&DS_F_ABL_PREV ogm.&DS_F_SF_PREV;
      run;
      proc sort data=factor_coll_prev nodupkey; by c_obg c_obl; run;
      proc sql;
        create table snapshot_prev as
        select a.*
        from base_snapshot_prev a
        left join factor_coll_prev b
          on a.c_obg=b.c_obg and a.c_obl=b.c_obl;
      quit;
    %end;
    %else %do;
      data snapshot_prev; set base_snapshot_prev; run;
    %end;

    /* Utilization prev */
    data snapshot_prev;
      set snapshot_prev;
      if tfc_face_amt>0 then utilization = tfc_curr_bal / tfc_face_amt;
      else utilization = .;
      format f_uplddt yymmdd10.;
    run;

    /* Trend: exposure-weighted utilization by risk over time (prev + curr) */
    data trend_union;
      set snapshot_prev(in=p) kevin_snapshot_cur(in=c);
      length src $5;
      src = ifc(p,'prev','curr');
      qtr = cats(put(year(f_uplddt),4.), 'Q', put(qtr(f_uplddt),1.));
    run;

    proc sql;
      create table trend_util_by_risk as
      select qtr,
             tfc_rsk_grd,
             sum(coalesce(tfc_curr_bal,0)) / nullif(sum(coalesce(tfc_face_amt,0)),0) as util_exp_weighted
      from trend_union
      where tfc_face_amt>0
      group by qtr, tfc_rsk_grd
      order by qtr, tfc_rsk_grd;
    quit;

    /* Migration: link same instrument across quarters by c_obg+c_obl (fallback to account id) */
    proc sql;
      create table migrate_pairs as
      select  p.c_obg, p.c_obl,
              coalesce(p.TFC_Account_Nbr, ' ') as acct_prev length=40,
              p.tfc_rsk_grd as r_prev,
              p.utilization as u_prev,
              c.tfc_rsk_grd as r_curr,
              c.utilization as u_curr
      from snapshot_prev p
      inner join kevin_snapshot_cur c
        on p.c_obg=c.c_obg and p.c_obl=c.c_obl;
    quit;

    /* Migration matrix with utilization at t and delta */
    proc sql;
      create table pivot_migration as
      select r_prev, r_curr,
             count(*) as n_facilities,
             mean(u_curr) as util_curr_mean,
             mean(u_curr - u_prev) as util_delta_mean
      from migrate_pairs
      group by r_prev, r_curr
      order by r_prev, r_curr;
    quit;
  %end;
%mend;
%build_prev_if_available;

/* -------------- Optional: CSV exports -------------- */
%macro export(ds, name);
  %if %length(&OUT_PATH.) %then %do;
    proc export data=&ds outfile="&OUT_PATH./&name..csv" dbms=csv replace; putnames=yes; run;
  %end;
%mend;

%export(kevin_snapshot_cur,              kevin_snapshot_current);
%export(pivot_util_by_risk,              util_by_risk);
%export(pivot_util_by_risk_collat,       util_by_risk_collateral);
%export(pivot_util_by_fac_risk,          util_by_facilitytype_risk);
%export(pivot_util_lgd_overlay,          util_lgd_overlay);
%export(pivot_util_by_vintage,           util_by_vintage);
%export(trend_util_by_risk,              util_trend_by_risk);
%export(pivot_migration,                 util_migration_matrix);
