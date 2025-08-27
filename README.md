/* ====== SETUP ====== */
options mprint mlogic symbolgen;
libname ogm "/sasdata/mrmg1";

/* Edit these if names/paths differ */
%let DS_ALL   = ogm.all_table_2503_1;         /* instrument snapshot */
%let DS_F_ABL = ogm.abl_lgd_factor_2503_abl;  /* LGD factor ABL */
%let DS_F_SF  = ogm.abl_lgd_factor_2503_sf;   /* LGD factor SF  */
%let OUT_CSV  = /sasdata/mrmg1/kevin_snapshot_simple.csv;

/* Helper: check whether a variable exists */
%macro hasvar(ds, var);
  %local dsid pos rc;
  %let dsid=%sysfunc(open(&ds));
  %if &dsid %then %do;
    %let pos=%sysfunc(varnum(&dsid,&var));
    %let rc=%sysfunc(close(&dsid));
    %sysfunc(ifc(&pos>0,1,0))
  %end;
  %else 0
%mend;

/* ====== 1) Base snapshot (only essentials, no collateral yet) ====== */
data _base0;
  set &DS_ALL;

  length LOB $3 Facility_Type $60 Risk_Rating $20 LGD_Rating $10;

  /* LOB: use SF flag when available; otherwise infer from LOB name */
  if nmiss(SF)=0 then LOB = ifc(SF=1,'SF','ABL');
  else do;
    LOB = '';
    if not missing(STI_lob_nm) then do;
      if index(upcase(STI_lob_nm),'STRUCTURED')>0 then LOB='SF';
      else if index(upcase(STI_lob_nm),'ABL')>0 then LOB='ABL';
    end;
  end;

  Facility_Type = coalescec(strip(STI_sub_lob_nm), strip(STI_lob_nm));
  Risk_Rating   = strip(TFC_rsk_grd);
  LGD_Rating    = strip(final_lgd_grade);
  Commitment    = tfc_face_amt;
  Drawn         = tfc_curr_bal;

  /* Utilization: only if commitment > 0; cap to [0,1] */
  if Commitment>0 then Util_fac = max(0, min(1, Drawn/Commitment));
  else Util_fac = .;

  keep LOB Facility_Type Risk_Rating LGD_Rating
       Commitment Drawn Util_fac
       c_obg c_obl TFC_Account_Nbr;
run;

/* ====== 2) Collateral from factor tables (ABL + SF) ====== */
/* Build views that always provide c_obg, c_obl, collateralclass, dt (dt=. if absent) */
%let HAS_DT_ABL = %hasvar(&DS_F_ABL, dt);
%let HAS_DT_SF  = %hasvar(&DS_F_SF,  dt);

%macro make_view(src, viewnm, hasdt);
  %if &hasdt=1 %then %do;
    proc sql; create view &viewnm as
      select c_obg, c_obl, collateralclass, dt from &src; quit;
  %end;
  %else %do;
    proc sql; create view &viewnm as
      select c_obg, c_obl, collateralclass, . as dt from &src; quit;
  %end;
%mend;

%make_view(&DS_F_ABL, F_ABL_V, &HAS_DT_ABL);
%make_view(&DS_F_SF,  F_SF_V,  &HAS_DT_SF);

data _fact_all;
  set F_ABL_V F_SF_V;
run;

/* Choose one collateral row per facility: latest dt if present */
proc sort data=_fact_all;
  by c_obg c_obl descending dt;
run;

data _coll_dom;
  set _fact_all;
  by c_obg c_obl;
  if first.c_obl then output;
run;

/* ====== 3) Join collateral into base (factor-only) ====== */
proc sql;
  create table Kevin_Snapshot as
  select b.*,
         f.collateralclass as Collateral_Class
  from _base0 b
  left join _coll_dom f
    on b.c_obg=f.c_obg and b.c_obl=f.c_obl;
quit;

/* ====== 4) Export for Excel ====== */
proc export data=Kevin_Snapshot
  outfile="&OUT_CSV" dbms=csv replace;
  putnames=yes;
run;
