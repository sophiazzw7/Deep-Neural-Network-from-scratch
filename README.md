/*-------------------------------------------------------------*/
/* Step 0: Subset to ET2 frequency data                        */
/*   Assumes ops.ef_dataset has:                               */
/*   - frequency (count)                                       */
/*   - Basel Event Type Level 1 (to filter ET2, if needed)     */
/*-------------------------------------------------------------*/

data freq_et2;
    set ops.ef_dataset;
    /* If needed, filter ET2 here; otherwise remove this IF */
    if strip('Basel Event Type Level 1'n) = "ET2 - External Fraud";
run;

/*-------------------------------------------------------------*/
/* Step 1: Fit NB model via PROC COUNTREG                      */
/*   model: frequency ~ Intercept only, dist=NEGBIN            */
/*-------------------------------------------------------------*/
proc countreg data=freq_et2;
    model frequency = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms;
run;

/* Extract intercept and alpha */
proc sql noprint;
    select estimate into :b0  trimmed
    from nb_parms
    where upcase(parameter) = 'INTERCEPT';

    select estimate into :alpha trimmed
    from nb_parms
    where upcase(parameter) like '%ALPHA%';

    /* Get sample size */
    select count(*) into :N_obs trimmed
    from freq_et2;
quit;

/* Convert to NB(p,r) form used by CDF/ RAND */
%let mu_hat = %sysevalf(exp(&b0));          /* mean */
%let r_hat  = %sysevalf(1/&alpha);          /* dispersion / #successes */
%let p_hat  = %sysevalf(&r_hat/(&r_hat + &mu_hat));  /* success prob */

/*-------------------------------------------------------------*/
/* Step 2: Compute CvM statistic on the observed data          */
/*-------------------------------------------------------------*/

/* 2a. Add model CDF to each observation and sort */
proc sort data=freq_et2 out=freq_sorted;
    by frequency;
run;

data freq_cdf;
    set freq_sorted;
    by frequency;
    retain i 0;
    i + 1;
    n = &N_obs;
    F_emp = (i - 0.5) / n;  /* empirical CDF at this rank */
    F_mod = cdf("NEGBINOMIAL", frequency, &p_hat, &r_hat);
    diff = F_mod - F_emp;
    diff2 = diff*diff;
run;

/* 2b. Sum squared differences to get CvM statistic */
proc means data=freq_cdf noprint;
    var diff2;
    output out=Obs_CvM (drop=_:) sum = W2_obs;
run;

proc print data=Obs_CvM label noobs;
    label W2_obs = "Observed Cramer–von Mises Statistic (W^2)";
run;
