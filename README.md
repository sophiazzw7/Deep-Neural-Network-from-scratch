/*=============================================================
   Step 1: Base instrument-level snapshot from All_table
   (this already includes LOB, balances, risk grades, LGD grades)
===============================================================*/

data base_snapshot;
    set new_ogm.all_table_2503_1;   /* replace with current quarter dataset */
    keep 
        TFC_Account_Nbr
        c_obg c_obl customerid

        /* Facility / LOB info */
        STI_lob_nm STI_sub_lob_nm business_group

        /* Commitment / Exposure */
        tfc_face_amt tfc_curr_bal

        /* Borrower Risk Rating */
        tfc_rsk_grd

        /* LGD ratings */
        model_lgd_grade final_lgd_grade final_lgd_perc

        /* Vintage / Age */
        f_uplddt
    ;
run;


/*=============================================================
   Step 2: Bring in collateral classification 
   (join Asset_Collateral_product → adds collateralclass, controltypr, valuationmethod)
===============================================================*/

proc sql;
    create table snapshot_with_collat as
    select a.*,
           b.collateralclass,
           b.controltypr,
           b.valuationmethod
    from base_snapshot a
    left join new_ogm.asset_collateral_product b
      on a.c_obg = b.c_obg
     and a.c_obl = b.c_obl
    ;
quit;


/*=============================================================
   Step 3: Add Utilization Indicator
===============================================================*/

data snapshot_final;
    set snapshot_with_collat;
    if tfc_face_amt > 0 then utilization = tfc_curr_bal / tfc_face_amt;
    else utilization = .;
run;


/*=============================================================
   Step 4: Pivot – Utilization vs Risk Rating
===============================================================*/

proc sql;
    create table util_by_risk as
    select tfc_rsk_grd,
           count(*) as n_facilities,
           mean(utilization) as avg_utilization
    from snapshot_final
    group by tfc_rsk_grd
    ;
quit;


/*=============================================================
   Step 5: Pivot – Utilization vs Risk Rating x Collateral Class
===============================================================*/

proc sql;
    create table util_by_risk_collat as
    select tfc_rsk_grd,
           collateralclass,
           count(*) as n_facilities,
           mean(utilization) as avg_utilization
    from snapshot_final
    group by tfc_rsk_grd, collateralclass
    ;
quit;


/*=============================================================
   Step 6: (Optional) Time series – Utilization vs Rating Migration
   If you have quarterly snapshots (multiple All_table_xx), 
   stack them and group by f_uplddt.
===============================================================*/

data all_qtr;
    set new_ogm.all_table_2403_1
        new_ogm.all_table_2503_1;   /* add more quarters as needed */
    keep TFC_Account_Nbr tfc_face_amt tfc_curr_bal tfc_rsk_grd f_uplddt;
run;

data all_qtr;
    set all_qtr;
    if tfc_face_amt > 0 then utilization = tfc_curr_bal / tfc_face_amt;
run;

proc sql;
    create table util_trend as
    select f_uplddt format=yymmn6.,
           tfc_rsk_grd,
           mean(utilization) as avg_utilization
    from all_qtr
    group by f_uplddt, tfc_rsk_grd
    order by f_uplddt, tfc_rsk_grd
    ;
quit;
