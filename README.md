/* 1. Performance Settings: Turn OFF Graphics and Results Window */
/* This is crucial. Without this, SAS tries to draw 1000 plots. */
ods graphics off;
ods exclude all;
ods noresults;

/* 2. Generate 1000 Bootstrap Samples */
/* Note: If your dataset is huge, reduce 'reps=1000' to 100 to test first */
proc surveyselect data=severity_et2_10k out=boot_samples seed=12345
     method=urs samprate=1 reps=1000;
run;

/* 3. Fit Truncated Lognormal to Each Replicate */
/* We use the exact options from your screenshot (maxiter=2000 tech=newrap) */
proc severity data=boot_samples print=none;
    by Replicate;
    loss gross_loss / lefttruncated=10000;
    dist logn;
    
    /* Use Developer's optimization settings to ensure convergence */
    nloptions maxiter=2000 tech=newrap;
    
    /* Save only the parameter estimates */
    ods output ParameterEstimates=boot_parms;
run;

/* 4. Restore Settings: Turn ON Graphics and Results */
ods graphics on;
ods exclude none;
ods results;

/* 5. Calculate and Print 95% Confidence Intervals */
proc univariate data=boot_parms noprint;
    class Parameter;
    var Estimate;
    output out=bootstrap_ci 
           pctlpre=CI_ 
           pctlpts=2.5, 97.5
           mean=MeanEstimate;
run;

title "Bootstrap 95% Confidence Intervals (Truncated Lognormal)";
proc print data=bootstrap_ci;
run;
