/* ----- Step 1: bootstrap resamples from the *actual* data ----- */
proc surveyselect data=frequency_et2
   method=urs                /* with replacement */
   samprate=1                /* same expected n as original */
   outhits                   /* keep duplicates */
   reps=500                  /* number of bootstrap reps; adjust as needed */
   seed=20241115
   out=boot;
run;

/* ----- Step 2: fit NB to each replicate ----- */
ods output ParameterEstimates=boot_parms;
proc countreg data=boot;
   by replicate;
   model frequency = / dist=NEGBIN;
run;
quit;

/* ----- Step 3: reshape to one row per replicate with b0 and alpha ----- */
data boot_parms_wide;
   set boot_parms;
   by replicate;
   retain b0 alpha;
   if first.replicate then do;
      b0 = .;
      alpha = .;
   end;

   if upcase(parameter)='INTERCEPT' then b0 = estimate;
   else if upcase(parameter) like '%ALPHA%' then alpha = estimate;

   if last.replicate then output;
   keep replicate b0 alpha;
run;

/* Drop reps where the refit failed or alpha nonpositive */
data boot_params_nb;
   set boot_parms_wide;
   if missing(b0) then delete;
   if missing(alpha) or alpha <= 0 then delete;

   mu = exp(b0);
   r  = 1/alpha;
   p  = r / (r + mu);
run;

/* Quick sanity check: how many usable bootstrap reps? */
proc sql;
  select count(*) as n_boot_usable from boot_params_nb;
quit;

/* ----- Step 4: percentile CIs from bootstrap distribution ----- */
proc univariate data=boot_params_nb noprint;
   var mu r p alpha b0;
   output out=NB_Boot_CIs
      pctlpre=mu_ r_ p_ alpha_ b0_
      pctlpts=2.5 97.5;
run;

data NB_Boot_Summary;
   length Param $12;
   set NB_Boot_CIs;

   Param='mu (mean)';  Estimate=&mu_hat;  LCL=mu_2_5;    UCL=mu_97_5;    output;
   Param='alpha';      Estimate=&alpha;   LCL=alpha_2_5; UCL=alpha_97_5; output;
   Param='r=1/alpha';  Estimate=&r_hat;   LCL=r_2_5;     UCL=r_97_5;     output;
   Param='p';          Estimate=&p_hat;   LCL=p_2_5;     UCL=p_97_5;     output;

   keep Param Estimate LCL UCL;
run;

title "Bootstrap Confidence Intervals for NB Parameters";
proc print data=NB_Boot_Summary noobs label;
   label Estimate='Point' LCL='Boot 2.5%' UCL='Boot 97.5%';
run; title;
