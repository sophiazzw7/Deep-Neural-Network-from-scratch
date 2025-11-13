/* 1) Fit NB and capture estimates + SEs */
ods output ParameterEstimates=nb_parms;
proc countreg data=frequency_et2;
  model frequency = / dist=NEGBIN;
run; quit;

/* 2) Make a one-row table with b0, seb0, alpha, sealpha */
proc sql;
  create table nb_coefs as
  select
    max(case when upcase(parameter)='INTERCEPT' then estimate end) as b0,
    max(case when upcase(parameter)='INTERCEPT' then stderr   end) as seb0,
    max(case when upcase(parameter) like '%ALPHA%' then estimate end) as alpha,
    max(case when upcase(parameter) like '%ALPHA%' then stderr   end) as sealpha
  from nb_parms;
quit;

/* Sanity check: ensure we have values */
proc print data=nb_parms(obs=10); title "COUNTREG ParameterEstimates"; run;
proc print data=nb_coefs; title "Extracted NB Coefficients"; run;

/* 3) Compute point estimates + 95% Wald CIs in data step (no macros) */
data NB_Wald_CIs;
  length Param $12;
  set nb_coefs;
  if nmiss(b0,seb0,alpha,sealpha)>0 then do;
    putlog "ERROR: Missing NB parameters or SEs. Check nb_parms.";
    stop;
  end;

  z = 1.96;

  /* Point estimates */
  mu_hat = exp(b0);
  r_hat  = 1/alpha;
  p_hat  = r_hat/(r_hat + mu_hat);

  /* Intercept CI -> mu CI */
  b0L = b0 - z*seb0;  b0U = b0 + z*seb0;
  muL = exp(b0L);     muU = exp(b0U);

  /* Alpha CI -> r CI (guard alpha > 0) */
  aL  = alpha - z*sealpha;
  aU  = alpha + z*sealpha;
  if aL <= 0 then aL = 1e-6;     /* numeric guard */
  if aU <= 0 then aU = 1e-6;

  rL  = 1/aU;                    /* monotone transform of endpoints */
  rU  = 1/aL;

  /* p CI via conservative endpoints (mu and r move adversely) */
  pL  = rL/(rL + muU);
  pU  = rU/(rU + muL);

  /* Assemble tidy output (4 rows) */
  Param='mu (mean)';  Estimate=mu_hat;  LCL=muL;  UCL=muU;  output;
  Param='alpha';      Estimate=alpha;   LCL=aL;   UCL=aU;   output;
  Param='r=1/alpha';  Estimate=r_hat;   LCL=rL;   UCL=rU;   output;
  Param='p';          Estimate=p_hat;   LCL=pL;   UCL=pU;   output;

  keep Param Estimate LCL UCL;
run;

title "Negative Binomial Wald Confidence Intervals";
proc print data=NB_Wald_CIs noobs label;
  label Estimate='Point' LCL='95% LCL' UCL='95% UCL';
run; title;
