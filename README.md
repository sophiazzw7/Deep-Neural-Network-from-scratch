/* 1. Turn off output to prevent crashing/freezing */
ods graphics off;
ods exclude all;
ods noresults;

/* 2. Generate 1000 Bootstrap Samples */
proc surveyselect data=severity_simulation_et2 out=boot_samples seed=12345
     method=urs samprate=1 reps=1000;
run;

/* 3. Fit Mixture Model to Each Replicate */
/* Exact syntax from your screenshot */
proc fmm data=boot_samples tech=newrap maxit=5000 gconv=0;
    by Replicate;
    
    /* Component 1: Custom Truncated Exponential */
    model y = / dist=TruncExpo(10000, .);
    
    /* Component 2: Lognormal */
    model + / dist=LogNormal;
    
    ods output ParameterEstimates=boot_parms;
run;

/* 4. Restore output settings */
ods graphics on;
ods exclude none;
ods results;

/* 5. Calculate CI for both components */
proc univariate data=boot_parms noprint;
    class Component Parameter; /* Groups by Model 1 vs Model 2 */
    var Estimate;
    output out=bootstrap_ci 
           pctlpre=CI_ 
           pctlpts=2.5, 97.5
           mean=MeanEst;
run;

title "Bootstrap CI for Mixture Model Parameters";
proc print data=bootstrap_ci;
run;
