/* Fit NB first (as you already do) */
proc countreg data=ops.edpm_dataset;
    model frequency = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms;
quit;

proc sql noprint;
    select estimate into :b0 trimmed
    from nb_parms
    where upcase(Parameter) = 'INTERCEPT';

    select estimate into :alpha trimmed
    from nb_parms
    where upcase(Parameter) like '%ALPHA%';
quit;

/* MLE-based NB parameters */
%let mu_hat = %sysfunc(exp(&b0));            /* mean */
%let r_hat  = %sysevalf(1/(&alpha));         /* dispersion (number of successes) */
%let p_hat  = %sysevalf(&r_hat / (&r_hat + &mu_hat));   /* success prob */

proc iml;

start AD_NB(x, p, r);
    /* Anderson–Darling statistic for general CDF F */
    call sort(x, 1);
    n  = nrow(x);
    F  = j(n,1,.);

    do i = 1 to n;
        F[i] = cdf("NEGBINOMIAL", x[i], p, r);

        /* Clamp away from 0 and 1 to avoid log(0) */
        if F[i] < 1e-12 then F[i] = 1e-12;
        if F[i] > 1-1e-12 then F[i] = 1-1e-12;
    end;

    idx = T(1:n);
    A2  = -n - (1/n) * sum( (2*idx-1)#( log(F) + log(1 - F[n+1-idx]) ) );
    return( A2 );
finish;

/* Read your counts */
use ops.edpm_dataset;
    read all var {frequency} into y;
close;

n      = nrow(y);
p      = &p_hat;
r      = &r_hat;
mu     = &mu_hat;

/* Observed statistic */
A2_obs = AD_NB(y, p, r);

/* Parametric bootstrap */
nBoot  = 10000;                 /* increase for stability */
call randseed(12345);

A2_boot = j(nBoot,1,.);

do b = 1 to nBoot;
    /* generate a NB sample of size n */
    yb = j(n,1,.);
    call randgen(yb, "NEGBINOMIAL", p, r);
    A2_boot[b] = AD_NB(yb, p, r);
end;

/* Bootstrap p-value (upper tail) */
pval = (1 + sum(A2_boot >= A2_obs)) / (nBoot + 1);

/* One-row summary dataset */
results = A2_obs || pval || p || r || mu || nBoot;
create AD_NB_Result from results
    [colname={"A2_obs" "pval" "p_hat" "r_hat" "mu_hat" "B"}];
append from results;
close AD_NB_Result;

/* Optional: save the bootstrap distribution for diagnostics */
create AD_NB_Bootstrap from A2_boot[colname={"A2_boot"}];
append from A2_boot;
close AD_NB_Bootstrap;

quit;

/* Pretty print the summary */
proc print data=AD_NB_Result label noobs;
    label A2_obs = "AD Statistic (Observed)"
          pval   = "Bootstrap p-value"
          p_hat  = "NB p (success prob)"
          r_hat  = "NB r (dispersion)"
          mu_hat = "Mean (mu)"
          B      = "Bootstrap reps";
run;

/* Optional: check bootstrap distribution to see if p-value makes sense */
proc means data=AD_NB_Bootstrap min p25 p50 p75 max;
    var A2_boot;
run;
