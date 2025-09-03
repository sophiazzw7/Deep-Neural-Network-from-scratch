2.1 Assign libs per quarter (v2.1 by default; 2025Q1 → v2.2)
libname L2023Q4 "/sasdata/mrmg1/MOD1497/2023Q4_v2.1";
libname L2024Q1 "/sasdata/mrmg1/MOD1497/2024Q1_v2.1";
libname L2024Q2 "/sasdata/mrmg1/MOD1497/2024Q2_v2.1";
libname L2024Q3 "/sasdata/mrmg1/MOD1497/2024Q3_v2.1";
libname L2024Q4 "/sasdata/mrmg1/MOD1497/2024Q4_v2.1";
libname L2025Q1 "/sasdata/mrmg1/MOD1497/2025Q1_v2.2";

2.2 Discover the actual table names (copy results from the log)
proc sql; 
  create table DISC as
  select libname, memname
  from sashelp.vtable
  where libname in ("L2023Q4","L2024Q1","L2024Q2","L2024Q3","L2024Q4","L2025Q1");
quit;

title "Tables that look like ALL / FACTOR / JOIN1497";
proc print data=DISC(where=(lowcase(memname) contains "all_table"
                         or lowcase(memname) contains "lgd_factor"
                         or lowcase(memname) contains "join_1497_final_res")) noobs;
  var libname memname;
run;
title;


From the printout, identify one all_table* per lib (quarter). Suppose you find:

L2023Q4.all_table_2023Q4, L2024Q1.all_table_2024Q1, …, L2025Q1.all_table_2025Q1

2.3 Build a simple union with override flag (no macros)
proc datasets lib=work nolist; delete OV_ALL; quit;

/* Repeat this DATA step block for each quarter you actually have. Keep it simple. */
data Q2023Q4;
  set L2023Q4.all_table_2023Q4;  /* <= replace with your exact name */
  length OGM_Qtr $7 segment $3;
  OGM_Qtr="2023Q4";
  if ^missing(SF) then segment=ifc(SF=1,"SF","ABL"); else segment="";
  id_fac = coalescec(strip(TFC_Account_Nbr_New), strip(TFC_Account_Nbr));
  asof_dt = ifn(missing(f_uplddt), ., datepart(f_uplddt));
  format asof_dt date9.;
  lgd_model = strip(model_lgd_grade);
  lgd_final = strip(final_lgd_grade);
  override_ind = (not missing(lgd_model) and not missing(lgd_final) and lgd_model ne lgd_final);
  keep OGM_Qtr segment id_fac asof_dt lgd_model lgd_final override_ind;
run;

data Q2024Q1; set L2024Q1.all_table_2024Q1; length OGM_Qtr $7 segment $3;
  OGM_Qtr="2024Q1";
  if ^missing(SF) then segment=ifc(SF=1,"SF","ABL"); else segment="";
  id_fac = coalescec(strip(TFC_Account_Nbr_New), strip(TFC_Account_Nbr));
  asof_dt = ifn(missing(f_uplddt), ., datepart(f_uplddt)); format asof_dt date9.;
  lgd_model=strip(model_lgd_grade); lgd_final=strip(final_lgd_grade);
  override_ind=(not missing(lgd_model) and not missing(lgd_final) and lgd_model ne lgd_final);
  keep OGM_Qtr segment id_fac asof_dt lgd_model lgd_final override_ind;
run;

/* ...repeat similarly for 2024Q2, Q3, Q4 ... */

data Q2025Q1; set L2025Q1.all_table_2025Q1; length OGM_Qtr $7 segment $3;
  OGM_Qtr="2025Q1";
  if ^missing(SF) then segment=ifc(SF=1,"SF","ABL"); else segment="";
  id_fac = coalescec(strip(TFC_Account_Nbr_New), strip(TFC_Account_Nbr));
  asof_dt = ifn(missing(f_uplddt), ., datepart(f_uplddt)); format asof_dt date9.;
  lgd_model=strip(model_lgd_grade); lgd_final=strip(final_lgd_grade);
  override_ind=(not missing(lgd_model) and not missing(lgd_final) and lgd_model ne lgd_final);
  keep OGM_Qtr segment id_fac asof_dt lgd_model lgd_final override_ind;
run;

/* Union */
data OV_ALL; set Q2023Q4 Q2024Q1 /* Q2024Q2 Q2024Q3 Q2024Q4 */ Q2025Q1; run;

/* Dedup within quarter by facility (keep latest record by asof_dt) */
proc sort data=OV_ALL; by OGM_Qtr id_fac asof_dt; run;
data OV_ALL; set OV_ALL; by OGM_Qtr id_fac asof_dt; if last.id_fac; run;

2.4 Core answers (override rate + repeat IDs)
/* Override rate by quarter and by segment */
proc sql;
  create table OV_RATE as
  select OGM_Qtr, coalescec(segment,"") as segment,
         count(*) as n_facilities,
         sum(override_ind) as n_overrides,
         calculated n_overrides / calculated n_facilities as override_rate format=percent8.2
  from OV_ALL
  group by OGM_Qtr, calculated segment
  order by OGM_Qtr, calculated segment;
quit;

title "Override Rate by Quarter and Segment";
proc print data=OV_RATE noobs; run;

/* Repeat overriders = same id_fac with overrides in 2+ quarters */
proc sql;
  create table REPEAT_FAC as
  select id_fac,
         count(distinct OGM_Qtr) as n_qtrs_seen,
         sum(override_ind) as n_quarters_with_override
  from OV_ALL
  where override_ind=1 and not missing(id_fac)
  group by id_fac
  having n_quarters_with_override >= 2
  order by n_quarters_with_override desc, n_qtrs_seen desc;
quit;

title "Facilities Overridden in 2+ Quarters";
proc print data=REPEAT_FAC(obs=50) noobs; run;
title;


This already answers: Are the same customers driving the breach? (Look at REPEAT_FAC.)

3) If you also want collateral (keep it simple)

Pick one factor table per quarter (the one that includes c_obg, c_obl, and either productid or f_fac, with collateralclass if available). Then:

/* Example: pick factor tables you have */
data FAC_ALL;
  set 
    L2023Q4.abl_lgd_factor_2023Q4
    L2024Q1.abl_lgd_factor_2024Q1
/*  L2024Q2.abl_lgd_factor_2024Q2
    L2024Q3.abl_lgd_factor_2024Q3
    L2024Q4.abl_lgd_factor_2024Q4 */
    L2025Q1.abl_lgd_factor_2025Q1
  ;
  length productid 8 collateralclass $40 id_fac $60;
  /* normalize productid */
  if vname(productid)='' then do;
    if vname(f_fac) ne '' then do;
      if vtype(f_fac)='C' then productid = input(strip(f_fac), best32.);
      else if vtype(f_fac)='N' then productid = f_fac;
    end;
  end;
  /* derive a facility key if present (optional) */
  id_fac = coalescec(strip(TFC_Account_Nbr_New), strip(TFC_Account_Nbr));
  keep c_obg c_obl productid collateralclass id_fac;
run;

/* Join OV_ALL to collateral by best available key.
   If your all_table lacks c_obg/c_obl reliably, use id_fac instead. */
proc sql;
  create table OV_ENRICH as
  select a.*, b.collateralclass
  from OV_ALL a
  left join FAC_ALL b
    on a.id_fac=b.id_fac;  /* or: a.c_obg=b.c_obg and a.c_obl=b.c_obl */
quit;

proc sql;
  create table OV_BY_COLL as
  select OGM_Qtr, coalescec(segment,"") as segment,
         coalescec(collateralclass,"(blank)") as collateralclass,
         count(*) as n_fac,
         sum(override_ind) as n_overrides,
         calculated n_overrides / calculated n_fac as override_rate format=percent8.2
  from OV_ENRICH
  group by OGM_Qtr, calculated segment, calculated collateralclass
  order by OGM_Qtr, calculated segment, calculated collateralclass;
quit;

title "Overrides by Collateral Class";
proc print data=OV_BY_COLL(obs=50) noobs; run;
title;
