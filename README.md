
Page for proc iml simulate NB:
https://documentation.sas.com/doc/en/imlug/15.2/imlug_langref_sect354.htm#imlug_langref028277


Got it—let’s test the RNG itself in a way Cheng will trust.

Below is SAS code you can run that:

Re-derives 
𝑝
p and 
𝑟
r from the COUNTREG output.

Uses PROC IML + CALL RANDGEN("NEGBINOMIAL", p, r) to simulate a large NB sample.

Compares empirical mean/variance and theoretical mean/variance.

Optionally checks the shape with a chi-square GOF.

You can paste this into a fresh program and just adjust the libname / dataset names if needed.

1. Get NB parameters from COUNTREG
/* Fit NB model exactly as in production */
proc countreg data=ops.ef_dataset;
    model frequency = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms;
quit;

/* Extract intercept and alpha into macro vars */
proc sql noprint;
    select estimate into :b0  trimmed from nb_parms
    where upcase(parameter) = 'INTERCEPT';

    select estimate into :alpha trimmed from nb_parms
    where upcase(parameter) like '%ALPHA%';
quit;

%put NOTE: Intercept=&b0  Alpha=&alpha;

/* Convert to NB(p,r) parametrization used by RANDGEN */
%let mu_hat = %sysevalf(exp(&b0));      /* mean */
%let r_hat  = %sysevalf(1/&alpha);      /* number of successes / dispersion */
%let p_hat  = %sysevalf(&r_hat / (&r_hat + &mu_hat));   /* success prob */

%put NOTE: mu_hat=&mu_hat  r_hat=&r_hat  p_hat=&p_hat;

2. Test the RANDGEN NEGBINOMIAL RNG
proc iml;
   /* Parameters from macro vars */
   p = &p_hat;
   r = &r_hat;

   /* Theoretical mean and variance for NB(r,p)
      (number of failures before r-th success) */
   mu_theo  = r * (1 - p) / p;
   var_theo = r * (1 - p) / (p##2);

   /* Simulate a LARGE NB sample */
   n = 1000000;        /* 1M draws; you can reduce if too slow */
   call randseed(12345);
   y = j(n,1,.);
   call randgen(y, "NEGBINOMIAL", p, r);

   /* Empirical mean and variance */
   mu_emp  = mean(y);
   var_emp = var(y);

   print p r,
         mu_theo var_theo,
         mu_emp  var_emp;

   /* Save simulated sample to a SAS dataset for further checks */
   create nb_sim from y[colname={'freq_sim'}];
   append from y;
   close nb_sim;
quit;


Interpretation:

If mu_emp ≈ mu_theo and var_emp ≈ var_theo (within Monte Carlo noise),
then CALL RANDGEN("NEGBINOMIAL", p, r) is behaving as a proper NB RNG.

Large discrepancies would indicate either:

a bug in our code (e.g., wrong 
𝑝
,
𝑟
p,r), or

a deeper issue with the RNG (very unlikely, but that is what Cheng is worried about).

3. Optional: Shape / GOF check on the simulated NB

This is just to reassure him that the distributional shape from IML looks like NB.

/* Compare simulated distribution to the model predictions */

proc means data=nb_sim n mean var maxdec=4;
    var freq_sim;
run;

/* Simple binned chi-square GOF against the fitted NB model */
proc freq data=nb_sim noprint;
    tables freq_sim / out=nb_bin_counts;
run;

/* nb_bin_counts now has observed counts for each value k.
   You can compute expected counts E_k from the NB pmf with p,r
   (in DATA step using PDF('NEGBINOMIAL', k, p, r) * N)
   and then compute a chi-square manually if you want. */
