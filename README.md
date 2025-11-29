/* 1. Create a clean parameter ID for each type of parameter */
data boot_param_all_long;
    set boot_param_all;
    length ParmID $40;

    /* You may need to adjust Model/Distribution names
       — check PROC CONTENTS of boot_param_all to confirm. */

    /* Exponential component */
    if Distribution = "Exponential" and Parameter = "Intercept" then
        ParmID = "Expo_Intercept";

    /* Lognormal component */
    else if Distribution = "LogNormal" and Parameter = "Intercept" then
        ParmID = "Lognorm_Intercept";
    else if Distribution = "LogNormal" and Parameter = "Scale" then
        ParmID = "Lognorm_Scale";

    /* Mixing probabilities – often come from ProbModel */
    else if Model = "ProbModel" and index(Parameter,"Logit") > 0 then do;
        if Component = 1 then ParmID = "Mix_LogitProb1";
        else if Component = 2 then ParmID = "Mix_LogitProb2";
    end;

    /* keep only rows we’ve labeled */
    if ParmID ne "";
run;

/* 2. Get bootstrap percentile CIs for each ParmID */
proc sort data=boot_param_all_long;
    by ParmID;
run;

proc univariate data=boot_param_all_long noprint;
    by ParmID;
    var Estimate;
    output out=sev_et2_param_CI
        pctlpts = 2.5 97.5
        pctlpre = CI_;
run;

proc print data=sev_et2_param_CI;
    title "Bootstrap 95% CIs for ET2 Severity Mixture Parameters";
run;
