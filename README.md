%macro boot_et2(B=500);

/* clear previous stacks */
proc datasets lib=work nolist; delete boot_all; quit;

%do b = 1 %to &B.;

    /* 1. Resample ET2 severity */
    proc surveyselect data=severity_et2_10k
        out=boot_sample method=urs samprate=1 outhits seed=&b;
    run;

    /* 2. Re-fit EXACT same model as developer */
    proc fmm data=boot_sample;
        model gross_loss = / dist=TruncExpo(&L_trunc.);
        model gross_loss = / dist=LogNormal;
        probmodel;
        ods output ParameterEstimates=boot_parm_&b;
    run;

    /* 3. Add replicate ID */
    data boot_parm_&b;
        set boot_parm_&b;
        replicate = &b;
    run;

    /* 4. stack */
    proc append base=boot_all data=boot_parm_&b force; run;

%end;

%mend;

%boot_et2(B=500);

/* Standardize parameter names */
data boot_all_clean;
    set boot_all;
    length ParmID $32;

    if Distribution="TruncExpo" and Parameter="Intercept" then ParmID="Expo_Intercept";
    else if Distribution="LogNormal" and Parameter="Intercept" then ParmID="Lognorm_Intercept";
    else if Distribution="LogNormal" and Parameter="Scale" then ParmID="Lognorm_Scale";
    else if Model="ProbModel" and index(Parameter,"Logit")>0 then do;
        if Component=1 then ParmID="Mix_Logit_1";
        else if Component=2 then ParmID="Mix_Logit_2";
    end;

    if ParmID ne "";
run;

proc sort data=boot_all_clean; by ParmID; run;

/* percentile CIs */
proc univariate data=boot_all_clean noprint;
    by ParmID;
    var Estimate;
    output out=CI_out
        pctlpts=2.5 97.5
        pctlpre=CI_;
run;

proc print data=CI_out;
run;
