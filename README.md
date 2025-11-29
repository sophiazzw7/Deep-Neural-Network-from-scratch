%let B = 500;   /* number of bootstrap samples */

proc datasets lib=work nolist;
    delete boot_param_all;
quit;

%macro boot_et2;

%do b = 1 %to &B.;

    /*------------------------------------------
      Step 1: simple resample with replacement
    ------------------------------------------*/
    data boot_sample;
        if 0 then set severity_et2_10k nobs=nobs;
        call streaminit(12345 + &b);

        do i = 1 to nobs;
            pt = ceil( rand("Uniform") * nobs );
            set severity_et2_10k point=pt;
            output;
        end;
        stop;
    run;

    /*------------------------------------------
      Step 2: fit same mixture as developer
    ------------------------------------------*/
    proc fmm data=boot_sample;
        model gross_loss = / dist=TruncExpo(10000);
        model gross_loss = / dist=LogNormal;
        probmodel;
        ods output ParameterEstimates=boot_param_&b;
    run;

    /* add replicate id and stack */
    data boot_param_&b;
        set boot_param_&b;
        Replicate = &b;
    run;

    proc append base=boot_param_all data=boot_param_&b force; run;

%end;

%mend;

%boot_et2;
2.2 Label parameters and compute bootstrap CIs
sas
Copy code
/* Give each parameter a clear ID */
data boot_param_all_clean;
    set boot_param_all;
    length ParmID $40;

    /* match on Distribution / Model / Component */
    if Distribution = "TruncExpo"  and Parameter="Intercept" then ParmID = "Expo_Intercept";
    else if Distribution = "LogNormal" and Parameter="Intercept" then ParmID = "Lognorm_Intercept";
    else if Distribution = "LogNormal" and Parameter="Scale"    then ParmID = "Lognorm_Scale";
    else if Model = "ProbModel" and index(Parameter,"Logit")>0 then do;
        if Component = 1 then ParmID = "Mix_Logit_1";
        else if Component = 2 then ParmID = "Mix_Logit_2";
    end;

    if ParmID ne "";
run;

/* Bootstrap percentile CIs with basic PROC MEANS */
proc sort data=boot_param_all_clean;
    by ParmID;
run;

proc means data=boot_param_all_clean n mean std p2_5 p97_5;
    by ParmID;
    var Estimate;
    title "Bootstrap CIs for ET2 Severity Mixture Parameters";
run;
