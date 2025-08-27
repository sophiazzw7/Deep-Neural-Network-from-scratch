/* ===== EDIT THIS to your desired target ===== */
%let OUT_XLSX = /sasdata/mrmg1/kevin_snapshot_simple.xlsx;
/* =========================================== */

/* 0) Confirm the dataset exists and row count */
%macro check_ds(ds);
  %if %sysfunc(exist(&ds)) %then %do;
    proc sql noprint; select count(*) into :_n from &ds; quit;
    %put NOTE: &ds exists with &_n rows.;
  %end;
  %else %do; %put ERROR: Dataset &ds not found. Stop.; %abort cancel; %end;
%mend;
%check_ds(work.Kevin_Snapshot);

/* 1) Try ODS EXCEL (robust on UNIX SAS) */
ods _all_ close;
ods excel file="&OUT_XLSX" options(sheet_name="Snapshot" embedded_titles='yes');
title "Kevin Snapshot";
proc print data=work.Kevin_Snapshot noobs; run;
ods excel close;
title;

/* 2) If ODS did not create the file, also write via XLSX engine */
libname xout xlsx "&OUT_XLSX";
data xout.Snapshot; set work.Kevin_Snapshot; run;
libname xout clear;

/* 3) Verify the exact file exists (server-side path) */
data _null_;
  if fileexist("&OUT_XLSX") then put "NOTE: XLSX created at &OUT_XLSX";
  else do;
    put "ERROR: Could not create &OUT_XLSX. Will try /tmp fallback.";
    call symputx('OUT_XLSX_FB','/tmp/kevin_snapshot_simple.xlsx','G');
  end;
run;

/* 4) Fallback to /tmp if needed */
%macro fallback;
  %if not %sysfunc(fileexist(&OUT_XLSX)) %then %do;
    ods _all_ close;
    ods excel file="&OUT_XLSX_FB" options(sheet_name="Snapshot");
    proc print data=work.Kevin_Snapshot noobs; run;
    ods excel close;

    libname xfb xlsx "&OUT_XLSX_FB";
    data xfb.Snapshot; set work.Kevin_Snapshot; run;
    libname xfb clear;

    data _null_;
      if fileexist("&OUT_XLSX_FB") then put "NOTE: Wrote XLSX to fallback: &OUT_XLSX_FB";
      else put "ERROR: Fallback write also failed. Check path/permissions.";
    run;
  %end;
%mend;
%fallback;
