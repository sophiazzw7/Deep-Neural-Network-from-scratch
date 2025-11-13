/* Fit (as you already do) */
proc countreg data=frequency_et2;
  model frequency = / dist=NEGBIN;
  ods output ParameterEstimates=nb_parms;
quit;

/* Pull estimates + SEs */
proc sql noprint;
  select estimate, stderr into :b0, :seb0  from nb_parms where upcase(parameter)='INTERCEPT';
  select estimate, stderr into :alpha, :sealpha from nb_parms where upcase(parameter) like '%ALPHA%';
quit;

/* Point estimates */
%let mu_hat = %sysfunc(exp(&b0));          /* mean */
%let r_hat  = %sysevalf(1/&alpha);         /* dispersion (number of successes) */
%let p_hat  = %sysevalf(&r_hat/(&r_hat+&mu_hat));

/* 95% Wald CI for Intercept, then transform to mu by exponentiation */
%let z = 1.96;
%let b0L = %sysevalf(&b0 - &z*&seb0);
%let b0U = %sysevalf(&b0 + &z*&seb0);
%let muL = %sysfunc(exp(&b0L));
%let muU = %sysfunc(exp(&b0U));

/* 95% Wald CI for Alpha, then transform to r = 1/alpha using endpoints (monotone) */
%let aL  = %sysevalf(&alpha - &z*&sealpha);
%let aU  = %sysevalf(&alpha + &z*&sealpha);
%if %sysevalf(&aL <= 0) %then %let aL = 1E-6;   /* guard against ≤0 */
%let rL  = %sysevalf(1/&aU);
%let rU  = %sysevalf(1/&aL);

/* Optional: delta-method symmetric CI for r
%let ser  = %sysevalf(&sealpha/(&alpha*&alpha));
%let rL_d = %sysevalf(&r_hat - &z*&ser);
%let rU_d = %sysevalf(&r_hat + &z*&ser);
*/

/* CI for p via transforming r and mu endpoints (conservative) */
%let pL = %sysevalf(&rL/(&rL + &muU));
%let pU = %sysevalf(&rU/(&rU + &muL));

data NB_Wald_CIs;
  length Param $12;
  Param='mu (mean)';  Estimate=&mu_hat;  LCL=&muL; UCL=&muU; output;
  Param='alpha';      Estimate=&alpha;   LCL=&aL;  UCL=&aU;  output;
  Param='r=1/alpha';  Estimate=&r_hat;   LCL=&rL;  UCL=&rU;  output;
  Param='p';          Estimate=&p_hat;   LCL=&pL;  UCL=&pU;  output;
run;

proc print data=NB_Wald_CIs noobs label; 
  label Estimate='Point' LCL='95% LCL' UCL='95% UCL';
run;
