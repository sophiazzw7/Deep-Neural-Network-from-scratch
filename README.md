/* ============ EDIT THIS PATH ONLY ============ */
%let OUT_XLSX = /sasdata/mrmg1/kevin_snapshot_simple.xlsx;
/* ============================================ */

/* 0) Sanity: show where SAS is running and that the output folder exists */
%put NOTE: Host=&SYSHOSTNAME  User=&SYSUSERID  OS=&SYSSCPL;
filename outdir "%sysfunc(filepath(work))"; /* just to show WORK path */
%put NOTE: WORK path = %sysfunc(pathname(work));
%let OUT_DIR = %substr(&OUT_XLSX,1,%length(&OUT_XLSX)-%length(%scan(&OUT_XLSX,-1,'/'))); /* crude dir pull */

/* Confirm Kevin_Snapshot exists and has rows */
%macro _chk_ds(ds);
  %if %sysfunc(exist(&ds)) %then %do;
    proc sql noprint; select count(*) into :_n from &ds; quit;
    %put NOTE: &ds exists with &_n observations.;
  %end;
  %else %do;
    %put ERROR: Dataset &ds not found. Stop.;
    %abort cancel;
  %end;
%mend;
%_chk_ds(work.Kevin_Snapshot);

/* 1) Try ODS EXCEL (most reliable on UNIX SAS) */
ods _all_ close;
ods excel file="&OUT_XLSX" options(sheet_name="Snapshot" embedded_titles='yes');
title "Kevin Snapshot";
proc print data=work.Kevin_Snapshot noobs; run;
ods excel close;

/* 1a) Verify the file now exists */
data _null_;
  length p $512;
  p = resolve('' " &OUT_XLSX " '');
  if fileexist("&OUT_XLSX") then put "NOTE: Wrote XLSX via ODS: &OUT_XLSX";
  else put "WARN: ODS EXCEL did not create &OUT_XLSX";
run;

/* 2) If not created, try LIBNAME XLSX engine */
%macro _fallback_xlsx;
  %if %sysfunc(fileexist("&OUT_XLSX"))=0 %then %do;
    libname xout xlsx "&OUT_XLSX";
    /* replace Snapshot sheet if it exists */
    proc datasets lib=xout nolist; delete Snapshot; quit;
    data xout.Snapshot; set work.Kevin_Snapshot; run;
    libname xout clear;
    data _null_; 
      if fileexist("&OUT_XLSX") then put "NOTE: Wrote XLSX via LIBNAME XLSX: &OUT_XLSX";
      else put "ERROR: Could not create &OUT_XLSX via either method.";
    run;
  %end;
%mend;
%_fallback_xlsx;

/* 3) List the target directory so you can see the file server-side */
filename lsdir pipe "ls -l %sysfunc(dequote(&OUT_DIR))";
data _null_;
  infile lsdir truncover;
  input line $char300.;
  putlog line;
run;
