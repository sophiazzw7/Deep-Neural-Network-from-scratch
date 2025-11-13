/* Reset output so we actually see results */
ods listing; ods results=on; options notes;

/* 1) Fit NB and capture estimates + SEs */
ods output ParameterEstimates=nb_parms;
proc countreg data=frequency_et2;
  model frequency = / dist=NEGBIN;
run; quit;

/* Show what we actually got back */
title "COUNTREG ParameterEstimates (first 10 rows)";
proc print data=nb_parms(obs=10); run; title;

/* 2) Make a one-row table with b0, seb0, alpha, sealpha (robust selectors) */
proc sql;
  create table nb_coefs as
  select
    max(case when upcase(parameter)='INTERCEPT' then estimate end) as b0,
    max(case when upcase(parameter)='INTERCEPT' then stderr   end) as seb0,
    /* alpha row name can be ALPHA or _ALPHA depending on SAS */
    max(case when upcase(parameter) like '%ALPHA%' then estimate end) as alpha,
    max(case when upcase(parameter) like '%ALPHA%' then stderr   end) as sealpha
  from nb_parms;
quit;

title "Extracted NB Coefficients";
proc print data=nb_coefs; run; title;

/* 3) Compute point estimates + 95% Wald CIs in a DATA step (never STOP; always print) */
data NB_Wald_CIs;
  length Param $14 ErrorNote $60;
  set nb_coefs;

  /* Flag any missing pieces but DON'T stop the step */
  ErrorNote = "";
  if nmiss(b0, seb0) then ErrorNote = catx("; ", ErrorNote, "Missing INTERCEPT/SE");
  if nmiss(alpha, sealpha) then ErrorNote = catx("; ", ErrorNote, "Missing ALPHA/SE");

  z = 1.96;

  /* Safeguards */
  if missing(b0) then b0 = .;
  if missing(seb0) then seb0 = .;
  if missing(alpha) then alpha = .;
  if missing(sealpha) then sealpha = .;

  /* Point estimates (may be . if inputs missing) */
  mu_hat = exp(b0);
  r_hat  = (alpha>0) ? 1/alpha : .;
  p_hat  = (mu_hat>=0 and r_hat>0) ? r_hat/(r_hat + mu_hat) : .;

  /* Intercept CI -> mu CI */
  b0L = b0 - z*seb0;  b0U = b0 + z*seb0;
  muL = exp(b0L);     muU = exp(b0U);

  /* Alpha CI -> r CI (guard alpha > 0 before invert) */
  aL  = alpha - z*sealpha;
  aU  = alpha + z*sealpha;
  if missing(aL) then aL = .;
  if missing(aU) then aU = .;
  if not missing(aL) and aL <= 0 then aL = 1e-6;   /* guard only if we have a value */
  if not missing(aU) and aU <= 0 then aU = 1e-6;

  rL  = (aU>0) ? 1/aU : .;
  rU  = (aL>0) ? 1/aL : .;

  /* p CI via conservative endpoints */
  pL  = (rL>0 and muU>=0) ? rL/(rL + muU) : .;
  pU  = (rU>0 and muL>=0) ? rU/(rU + muL) : .;

  /* Emit 4 tidy rows */
  Param='mu (mean)';     Estimate=mu_hat;  LCL=muL;  UCL=muU;  output;
  Param='alpha';         Estimate=alpha;   LCL=aL;   UCL=aU;   output;
  Param='r = 1/alpha';   Estimate=r_hat;   LCL=rL;   UCL=rU;   output;
  Param='p';             Estimate=p_hat;   LCL=pL;   UCL=pU;   output;

  keep Param Estimate LCL UCL ErrorNote;
run;

/* If nothing prints now, we know the fit didn't return usable rows */
title "Negative Binomial Wald Confidence Intervals";
proc print data=NB_Wald_CIs noobs label;
  label Estimate='Point' LCL='95% LCL' UCL='95% UCL' ErrorNote='Notes';
run; title;

/* Bonus: explicit row counts so EG doesn't just show a red X */
proc sql;
  select
    (select count(*) from nb_parms)   as rows_nb_parms,
    (select count(*) from nb_coefs)   as rows_nb_coefs,
    (select count(*) from NB_Wald_CIs) as rows_nb_wald_cis;
quit;
