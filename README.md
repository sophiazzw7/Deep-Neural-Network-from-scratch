/*-----------------------------------------------------------
  STEP 0: Subset ET2 frequency data
------------------------------------------------------------*/
data freq_et2;
    set ops.ef_dataset;
    if strip('Basel Event Type Level 1'n)
       = "ET2 - External Fraud";
run;

/*-----------------------------------------------------------
  STEP 1: Fit Negative Binomial model via COUNTREG
------------------------------------------------------------*/
proc countreg data=freq_et2;
    model frequency = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms_et2;
run;

/* Extract intercept and alpha */
proc sql noprint;
    select estimate into :b0  trimmed
    from nb_parms_et2
    where upcase(parameter) = 'INTERCEPT';

    select estimate into :alpha trimmed
    from nb_parms_et2
    where upcase(parameter) like '%ALPHA%';

    select count(*) into :N_et2 trimmed
    from freq_et2;
quit;

/* Convert to NB(p,r) */
%let mu_hat = %sysevalf(exp(&b0));           
%let r_hat  = %sysevalf(1/&alpha);           
%let p_hat  = %sysevalf(&r_hat / (&r_hat + &mu_hat)); 
