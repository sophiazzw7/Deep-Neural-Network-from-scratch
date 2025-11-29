/* 1. Generate 1000 Bootstrap Samples */
proc surveyselect data=severity_et2_10k out=boot_samples seed=12345
     method=urs samprate=1 reps=1000;
run;

/* 2. Fit Truncated Lognormal to Each Replicate */
proc severity data=boot_samples print=none;
    by Replicate;
    loss gross_loss / lefttruncated=10000;
    dist logn;
    ods output ParameterEstimates=boot_parms;
run;

/* 3. Calculate 95% Confidence Intervals */
proc univariate data=boot_parms noprint;
    class Parameter;
    var Estimate;
    output out=bootstrap_ci 
           pctlpre=CI_ 
           pctlpts=2.5, 97.5;
run;

proc print data=bootstrap_ci;
run;
