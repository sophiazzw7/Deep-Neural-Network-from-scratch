/*-----------------------------------------------------------
   1. Fit Poisson model using COUNTREG (parallel to NB version)
-----------------------------------------------------------*/
proc countreg data=ops.edpm_dataset;
    model frequency = / dist=POISSON;
    ods output ParameterEstimates = pois_parms;
quit;

/* Extract Poisson intercept parameter */
/* For Poisson log-link, mean = exp(intercept) */
proc sql noprint;
    select estimate into :b0 trimmed
    from pois_parms
    where upcase(Parameter) = 'INTERCEPT';
quit;

%let mu_hat = %sysfunc(exp(&b0));   /* Poisson mean */
%put NOTE: Estimated Poisson mean (mu_hat) = &mu_hat.;

/*-----------------------------------------------------------
   2. Anderson–Darling Statistic for Poisson
-----------------------------------------------------------*/
proc iml;

start AD_Pois(x, mu);
    call sort(x, 1);
    n  = nrow(x);
    F  = j(n,1,.);

    do i = 1 to n;
        F[i] = cdf("POISSON", x[i], mu);

        /* Clamp to avoid log(0) */
        if F[i] < 1e-12 then F[i] = 1e-12;
        if F[i] > 1-1e-12 then F[i] = 1-1e-12;
    end;

    idx = T(1:n);
    A2 = -n - (1/n) * sum( (2*idx-1)#( log(F) + log(1 - F[n+1-idx]) ) );
    return(A2);
finish;

/*-----------------------------------------------------------
   3. Read data and compute observed AD statistic
-----------------------------------------------------------*/
use ops.edpm_dataset;
    read all var {frequency} into y;
close;

n  = nrow(y);
mu = &mu_hat;
A2_obs = AD_Pois(y, mu);

/*-----------------------------------------------------------
   4. Parametric bootstrap (10,000 reps)
-----------------------------------------------------------*/
nBoot = 10000;
call randseed(12345);

A2_boot = j(nBoot,1,.);

do b = 1 to nBoot;
    yb = j(n,1,.);
    call randgen(yb, "POISSON", mu);
    A2_boot[b] = AD_Pois(yb, mu);
end;

/* Bootstrap p-value */
pval = (1 + sum(A2_boot >= A2_obs)) / (nBoot + 1);

/*-----------------------------------------------------------
   5. Output results
-----------------------------------------------------------*/
results = A2_obs || pval || mu || nBoot;

create AD_Pois_Result from results
    [colname={"A2_obs" "pval" "mu_hat" "B"}];
append from results;
close AD_Pois_Result;

create AD_Pois_Bootstrap from A2_boot[colname={"A2_boot"}];
append from A2_boot;
close AD_Pois_Bootstrap;

quit;

/*-----------------------------------------------------------
   6. Print summary
-----------------------------------------------------------*/
proc print data=AD_Pois_Result noobs label;
    label 
        A2_obs = "AD Statistic (Observed)"
        pval   = "Bootstrap p-value"
        mu_hat = "Poisson mean (mu)"
        B      = "Bootstrap reps";
run;

/* Optional: bootstrap distribution view */
proc means data=AD_Pois_Bootstrap min p25 p50 p75 max;
    var A2_boot;
run;
