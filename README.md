/* ===================== 0) USER PARAMS ===================== */
options mprint mlogic symbolgen nosource2;
%let ROOT  = /sasdata/mrmg1/MOD1497;                 /* parent folder containing 2024Q2_v2.1, etc. */
%let QLIST = 2023Q4 2024Q1 2024Q2 2024Q3 2024Q4 2025Q1;

/* default version for all quarters */
%let DEF_VERS = v2.1;
/* per-quarter override(s) */
%let VERS_2025Q1 = v2.2;

/* (Optional) Excel export path */
%let OUT_XLSX = /sasdata/mrmg1/MOD1497/outputs/override_cross_quarter.xlsx;

/* ===================== UTIL MACROS ===================== */
%macro exists(ds); %sysfunc(exist(&ds)) %mend;

%macro hasvar(ds, var);
  %local dsid vnum rc;
  %let dsid=%sysfunc(open(&ds,i));
  %if &dsid %then %do;
    %let vnum=%sysfunc(varnum(&dsid,%upcase(&var)));
    %let rc=%sysfunc(close(&dsid));
    %sysfunc(ifc(&vnum>0,1,0))
  %end;
  %else 0
%mend;

%macro qvers(q);
  %local _v;
  %let _v = &&VERS_&q;
  %if %length(&_v)=0 %then %let _v=&DEF_VERS;
  &_v
%mend;

/* LGD letter → ordinal for signed override severity (+upgrade / -downgrade) */
proc format;
  value $lgd_ord
    'A'=1  'B'=2  'C'=3  'D'=4  'E'=5  'F'=6  'G'=7  'H'=8  'I'=9  'J'=10
    'K'=11 'L'=12 'M'=13 'N'=14 'O'=15 'P'=16 'Q'=17 'R'=18 'S'=19 'T'=20 ;
run;

/* ===================== 1) LIBNAMES PER QUARTER ===================== */
%macro assign_libs;
  %local i q libpth libnm ver;
  %do i=1 %to %sysfunc(countw(&QLIST,%str( )));
    %let q   = %scan(&QLIST,&i,%str( ));
    %let ver = %qvers(&q);
    %let libpth = &ROOT./&q._&ver.;
    %let libnm  = L%sysfunc(compress(&q, , 'kad'));  /* e.g., L2025Q1 */

    libname &libnm "&libpth";
    %put NOTE: Assigned &libnm to &libpth ;
  %end;
%mend;
%assign_libs;

/* ===================== 2) DISCOVER DATASETS IN EACH QUARTER ===================== */
proc datasets lib=work nolist; delete DISC_ALL DISC_FAC DISC_J1497; quit;

%macro discover_for_lib(libref, qtr);
  proc sql noprint;
    create table _v_&libref as
    select libname, memname
    from sashelp.vtable
    where libname="&libref";

    create table _all_&libref as
    select "&qtr" as ogm_qtr length=7, "&libref" as lib length=12, memname
    from _v_&libref
    where lowcase(memname) contains "all_table";

    create table _fac_&libref as
    select "&qtr" as ogm_qtr length=7, "&libref" as lib length=12, memname
    from _v_&libref
    where lowcase(memname) contains "lgd_factor";

    create table _j_&libref as
    select "&qtr" as ogm_qtr length=7, "&libref" as lib length=12, memname
    from _v_&libref
    where lowcase(memname) contains "join_1497_final_res";
  quit;

  proc append base=DISC_ALL   data=_all_&libref force; run;
  proc append base=DISC_FAC   data=_fac_&libref force; run;
  proc append base=DISC_J1497 data=_j_&libref   force; run;

  proc datasets lib=work nolist; delete _v_&libref _all_&libref _fac_&libref _j_&libref; quit;
%mend;

%macro discover_all;
  %local i q libnm;
  %do i=1 %to %sysfunc(countw(&QLIST,%str( )));
    %let q=%scan(&QLIST,&i,%str( ));
    %let libnm = L%sysfunc(compress(&q, , 'kad'));
    %discover_for_lib(&libnm,&q);
  %end;
%mend;
%discover_all;

/* ===================== 3) BUILD OV_ALL FROM ALL_TABLES ===================== */
proc datasets lib=work nolist; delete OV_ALL _:; quit;

%macro build_OV_ALL;
  %local n i q lib mem dsn;
  proc sql noprint; select count(*) into :n from DISC_ALL; quit;
  %if &n=0 %then %do; %put ERROR: No ALL tables found under the specified folders.; %return; %end;

  data _list; set DISC_ALL; idx=_N_; run;

  %do i=1 %to &n;
    data _one; set _list; if idx=&i; run;
    proc sql noprint; select ogm_qtr, lib, memname into :q, :lib, :mem from _one; quit;
    %let dsn = &lib..&mem;

    %if %exists(&dsn) %then %do;
      data _one_&q;
        set &dsn;
        length segment $3 risk_grade $12 lgd_model $1 lgd_final $1
               id_fac $60 id_obgbl $120 OGM_Qtr $7;

        /* Segment flag if available */
        %if %hasvar(&dsn,SF) %then %do;
          if SF=1 then segment='SF'; else segment='ABL';
        %end;
        %else %do; segment=''; %end;

        risk_grade = strip(coalescec(TFC_rsk_grd,''));
        lgd_model  = strip(coalescec(model_lgd_grade,''));
        lgd_final  = strip(coalescec(final_lgd_grade,''));

        id_fac   = coalescec(strip(TFC_Account_Nbr_New), strip(TFC_Account_Nbr));
        id_obgbl = cats(strip(c_obg),'|',strip(c_obl));

        /* As-of date */
        %if %hasvar(&dsn,f_uplddt) %then %do;
          asof_dt = datepart(f_uplddt);
        %end;
        %else %do; asof_dt=.; %end;
        format asof_dt date9.;
        OGM_Qtr  = "&q";

        /* EAD components if present */
        %if %hasvar(&dsn,tfc_face_amt) %then %do; commit=tfc_face_amt; %end; %else %do; commit=.; %end;
        %if %hasvar(&dsn,tfc_curr_bal) %then %do; drawn =tfc_curr_bal; %end; %else %do; drawn =.; %end;

        /* Override indicator and severity */
        override_ind = (not missing(lgd_model) and not missing(lgd_final) and lgd_final ne lgd_model);
        if override_ind then override_sev = input(lgd_final,$lgd_ord.) - input(lgd_model,$lgd_ord.);

        keep segment risk_grade lgd_model lgd_final override_ind override_sev
             asof_dt OGM_Qtr commit drawn id_fac id_obgbl c_obg c_obl;
      run;

      proc sort data=_one_&q; by OGM_Qtr id_fac asof_dt; run;
      data one_&q; set _one_&q; by OGM_Qtr id_fac asof_dt; if last.id_fac then output; run;

      proc append base=OV_ALL data=one_&q force; run;
      proc datasets lib=work nolist; delete _one_&q one_&q _one; quit;
    %end;
    %else %put NOTE: &dsn not found; skipping.;
  %end;

  %if %exists(OV_ALL)=0 %then %do; %put ERROR: OV_ALL not created.; %return; %end;
%mend;
%build_OV_ALL;

/* ===================== 4) UNION FACTOR TABLES → FAC_ALL (for collateral) ===================== */
proc datasets lib=work nolist; delete FAC_ALL; quit;

%macro build_FAC_ALL;
  %local n i lib mem dsn;
  proc sql noprint; select count(*) into :n from DISC_FAC; quit;
  %if &n=0 %then %do; %put NOTE: No lgd_factor tables found; skipping collateral.; %return; %end;

  data _faclist; set DISC_FAC; idx=_N_; run;

  %do i=1 %to &n;
    data _one; set _faclist; if idx=&i; run;
    proc sql noprint; select lib, memname into :lib, :mem from _one; quit;
    %let dsn = &lib..&mem;

    %if %exists(&dsn) %then %do;
      data _f&i;
        set &dsn;
        length productid 8 collateralclass $40;

        /* Normalize productid: prefer productid; else f_fac (char or numeric) */
        if vname(productid) ne '' then productid = productid;
        else if vname(f_fac) ne '' then do;
          if vtype(f_fac)='C' then productid = input(strip(f_fac), best32.);
          else if vtype(f_fac)='N' then productid = f_fac;
        end;

        /* collateral class if present */
        if vname(collateralclass)='' then collateralclass=''; else collateralclass=strip(collateralclass);

        keep c_obg c_obl productid collateralclass;
      run;

      proc append base=FAC_ALL data=_f&i force; run;
      proc datasets lib=work nolist; delete _f&i _one; quit;
    %end;
  %end;
%mend;
%build_FAC_ALL;

/* ===================== 5) UNION JOIN_1497_FINAL_RES → J1497 (optional) ===================== */
proc datasets lib=work nolist; delete J1497; quit;

%macro build_J1497;
  %local n i lib mem dsn;
  proc sql noprint; select count(*) into :n from DISC_J1497; quit;
  %if &n=0 %then %do; %put NOTE: No join_1497_final_res tables found; skipping.; %return; %end;

  data _jlist; set DISC_J1497; idx=_N_; run;

  %do i=1 %to %sysfunc(attrn(%sysfunc(open(_jlist)),nobs));
    data _one; set _jlist; if idx=&i; run;
    proc sql noprint; select lib, memname into :lib, :mem from _one; quit;
    %let dsn = &lib..&mem;

    %if %exists(&dsn) %then %do;
      data _j&i;
        set &dsn;
        length id_fac $60 id_obgbl $120;

        id_fac   = coalescec(strip(TFC_Account_Nbr_New), strip(TFC_Account_Nbr));
        id_obgbl = cats(strip(c_obg),'|',strip(c_obl));

        /* Safely carry reason fields if present */
        %if %hasvar(&dsn,override_reason) %then %do; override_reason=override_reason; %end;
        %else %do; length override_reason 8; override_reason=.; %end;

        %if %hasvar(&dsn,reason_text) %then %do; reason_text=reason_text; %end;
        %else %do; length reason_text $200; reason_text=''; %end;

        %if %hasvar(&dsn,model_lgd_grade) %then %do; model_lgd_grade=model_lgd_grade; %end;
        %else %do; length model_lgd_grade $1; model_lgd_grade=''; %end;

        %if %hasvar(&dsn,final_lgd_grade) %then %do; final_lgd_grade=final_lgd_grade; %end;
        %else %do; length final_lgd_grade $1; final_lgd_grade=''; %end;

        keep id_fac id_obgbl c_obg c_obl override_reason reason_text model_lgd_grade final_lgd_grade;
      run;

      proc append base=J1497 data=_j&i force; run;
      proc datasets lib=work nolist; delete _j&i _one; quit;
    %end;
  %end;
%mend;
%build_J1497;

/* ===================== 6) ENRICH OV_ALL WITH COLLATERAL / JOIN1497 ===================== */
%macro enrich_all;
  %if %exists(FAC_ALL) %then %do;
    proc sql;
      create table OV_ENRICH as
      select a.*,
             b.productid,
             coalesce(a.collateralclass, b.collateralclass) as collateralclass length=40
      from OV_ALL as a
      left join FAC_ALL as b
        on a.c_obg=b.c_obg and a.c_obl=b.c_obl;
    quit;
  %end;
  %else %do;
    data OV_ENRICH; set OV_ALL; run;
  %end;

  %if %exists(J1497) %then %do;
    proc sql;
      create table OV_ENRICH as
      select e.*,
             j.override_reason,
             j.reason_text
      from OV_ENRICH e
      left join J1497 j
        on e.id_fac=j.id_fac
      ;
    quit;
  %end;
%mend;
%enrich_all;

/* ===================== 7) ANALYSIS TABLES ===================== */
proc sql;
  /* 7a) Override rate by quarter & segment */
  create table OV_RATE as
  select OGM_Qtr, coalescec(segment,'') as segment,
         count(*)                                  as n_facilities,
         sum(override_ind)                         as n_overrides,
         calculated n_overrides / calculated n_facilities as override_rate format=percent8.2,
         mean(override_sev)                        as avg_signed_severity
  from OV_ENRICH
  group by OGM_Qtr, calculated segment
  order by OGM_Qtr, calculated segment;

  /* 7b) Repeat overriders (same facility with overrides in 2+ quarters) */
  create table REPEAT_FAC as
  select id_fac,
         count(distinct OGM_Qtr)                   as n_qtrs_seen,
         sum(override_ind)                         as n_quarters_with_override,
         mean(override_sev)                        as mean_severity,
         max(abs(override_sev))                    as max_abs_severity
  from OV_ENRICH
  where override_ind=1 and not missing(id_fac)
  group by id_fac
  having n_quarters_with_override >= 2
  order by n_quarters_with_override desc, n_qtrs_seen desc;

  /* 7c) Overrides by collateral class (from factors, if available) */
  create table OV_BY_COLL as
  select OGM_Qtr,
         coalescec(segment,'') as segment,
         coalescec(collateralclass,'(blank)') as collateralclass,
         count(*)                        as n_fac,
         sum(override_ind)               as n_overrides,
         calculated n_overrides / calculated n_fac as override_rate format=percent8.2
  from OV_ENRICH
  group by OGM_Qtr, calculated segment, calculated collateralclass
  order by OGM_Qtr, calculated segment, calculated collateralclass;

  /* 7d) Direction by starting model grade */
  create table OV_BY_GRADE as
  select OGM_Qtr, coalescec(segment,'') as segment, lgd_model,
         sum(override_ind)                                         as n_overrides,
         mean(override_sev<0)                                      as pct_downgrade format=percent8.1,
         mean(override_sev>0)                                      as pct_upgrade  format=percent8.1
  from OV_ENRICH
  group by OGM_Qtr, calculated segment, lgd_model
  order by OGM_Qtr, calculated segment, lgd_model;

  /* 7e) Top 25 repeat facilities (with last known collateral/segment) */
  create table TOP25_REPEAT as
  select r.id_fac,
         max(e.collateralclass)     as collateralclass,
         max(e.segment)             as segment,
         max(e.OGM_Qtr)             as last_qtr_seen,
         r.n_qtrs_seen,
         r.n_quarters_with_override,
         r.mean_severity,
         r.max_abs_severity
  from REPEAT_FAC r
  left join OV_ENRICH e
    on r.id_fac=e.id_fac
  group by r.id_fac
  order by r.n_quarters_with_override desc, r.n_qtrs_seen desc;
quit;

/* ===================== 8) QUICK PRINTS (optional) ===================== */
title "Override Rate by Quarter & Segment";
proc print data=OV_RATE noobs; run;

title "Repeat Overriders (>=2 quarters)";
proc print data=REPEAT_FAC(obs=50) noobs; run;

title "Overrides by Collateral Class";
proc print data=OV_BY_COLL(obs=50) noobs; run;

title "Top 25 Repeat Facilities";
proc print data=TOP25_REPEAT noobs; run;

title;

/* ===================== 9) EXPORT TO EXCEL (optional) ===================== */
%macro export_xlsx;
  %if %length(&OUT_XLSX) %then %do;
    filename xout "&OUT_XLSX";
    proc export data=OV_RATE        outfile=xout dbms=xlsx replace; sheet="Override_Rate";        run;
    proc export data=REPEAT_FAC     outfile=xout dbms=xlsx replace; sheet="Repeat_Facilities";    run;
    proc export data=OV_BY_COLL     outfile=xout dbms=xlsx replace; sheet="By_Collateral";        run;
    proc export data=OV_BY_GRADE    outfile=xout dbms=xlsx replace; sheet="By_Grade";             run;
    proc export data=TOP25_REPEAT   outfile=xout dbms=xlsx replace; sheet="Top25_Repeat";         run;
    filename xout clear;
    %put NOTE: Results exported to &OUT_XLSX ;
  %end;
%mend;
%export_xlsx;
