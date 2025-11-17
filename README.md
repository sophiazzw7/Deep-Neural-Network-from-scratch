/*-----------------------------------------------------------
  0. Optional: clean previous work tables
-----------------------------------------------------------*/
proc datasets lib=work nolist;
    delete nb_parms AD_NB_Result AD_NB_Bootstrap;
quit;

/*-----------------------------------------------------------
  1. Fit NB frequency model exactly as developer did
-----------------------------------------------------------*/
libname ops "/sasdata/mrmg2/users/G07267/MOD13638_2025/Output";

proc countreg data=ops.ef_dataset;
    model frequency = / dist=NEGBIN;
    output out=countreg_frequency_pred_negbig pred=pred_negbin;
    ods output ParameterEstimates = nb_parms;   * added only to capture parameters;
quit;
title;

/*-----------------------------------------------------------
  2. Extract NB parameters from developer fit
-----------------------------------------------------------*/
proc sql noprint;
    select estimate into :b0 trimmed
    from nb_parms
    where upcase(Parameter) = 'INTERCEPT';

    select estimate into :alpha trimmed
    from nb_parms
    where upcase(Parameter) like '%ALPHA%';
quit;

/* Negative Binomial parameterization matching SAS:
   mean mu = exp(b0)
   variance = mu*(1 + alpha*mu)
   r = 1/alpha   (number of successes)
   p = r / (r + mu)  (success probability)
*/
%let mu_hat = %sysfunc(exp(&b0));                      /* mean */
%let r_hat  = %sysevalf(1/(&alpha));                   /* dispersion */
%let p_hat  = %sysevalf(&r_hat / (&r_hat + &mu_hat));  /* success prob */

/*-----------------------------------------------------------
  3. Anderson–Darling statistic function for NB
-----------------------------------------------------------*/
proc iml;

start AD_NB(x, p, r);
    call sort(x, 1);
    n  = nrow(x);
    F  = j(n,1,.);

    do i = 1 to n;
        F[i] = cdf("NEGBINOMIAL", x[i], p, r);

        /* clamp away from 0 and 1 to avoid log(0) */
        if F[i] < 1e-12 then F[i] = 1e-12;
        if F[i] > 1-1e-12 then F[i] = 1-1e-12;
    end;

    idx = T(1:n);
    A2  = -n - (1/n) * sum( (2*idx-1)#( log(F) + log(1 - F[n+1-idx]) ) );
    return( A2 );
finish;

/* Read the same frequency data used by developer */
use ops.ef_dataset;
    read all var {frequency} into y;
close;

n  = nrow(y);
p  = &p_hat;
r  = &r_hat;
mu = &mu_hat;

/* Observed AD statistic */
A2_obs = AD_NB(y, p, r);

/*-----------------------------------------------------------
  4. Parametric bootstrap for p-value
-----------------------------------------------------------*/
nBoot  = 10000;
call randseed(12345);

A2_boot = j(nBoot,1,.);

do b = 1 to nBoot;
    yb = j(n,1,.);
    call randgen(yb, "NEGBINOMIAL", p, r);
    A2_boot[b] = AD_NB(yb, p, r);
end;

/* Upper-tail bootstrap p-value */
pval = (1 + sum(A2_boot >= A2_obs)) / (nBoot + 1);

/*-----------------------------------------------------------
  5. Output AD results
-----------------------------------------------------------*/
results = A2_obs || pval || p || r || mu || nBoot;
create AD_NB_Result from results
    [colname={"A2_obs" "pval" "p_hat" "r_hat" "mu_hat" "B"}];
append from results;
close AD_NB_Result;

create AD_NB_Bootstrap from A2_boot[colname={"A2_boot"}];
append from A2_boot;
close AD_NB_Bootstrap;

quit;

/*-----------------------------------------------------------
  6. Print summary and bootstrap diagnostics
-----------------------------------------------------------*/
proc print data=AD_NB_Result label noobs;
    label A2_obs = "AD Statistic (Observed)"
          pval   = "Bootstrap p-value"
          p_hat  = "NB p (success prob)"
          r_hat  = "NB r (dispersion)"
          mu_hat = "Mean (mu)"
          B      = "Bootstrap reps";
run;

proc means data=AD_NB_Bootstrap min p25 p50 p75 max;
    var A2_boot;
run;
