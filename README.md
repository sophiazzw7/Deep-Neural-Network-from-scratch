/*===============================================================
  BOOTSTRAP STABILITY for Negative Binomial (intercept-only)
  - METHOD=NP  : Nonparametric bootstrap (resample original data)
  - METHOD=PAR : Parametric bootstrap (simulate from fitted NB)
  Outputs:
    - NB_Boot_Estimates : per-replicate parameter estimates
    - NB_Boot_CIs       : bootstrap percentile CIs
    - NB_Boot_Summary   : baseline estimate + SE + CI + RCIW
===============================================================*/
%macro nb_bootstrap_stability(ds=, var=frequency, B=1000, seed=20241113, method=NP);

  %local _method;
  %let _method=%upcase(&method);

  /*---- 0) Baseline NB fit to get point estimates ----*/
  ods output ParameterEstimates=nb_parms FitStatistics=nb_fit;
  proc countreg data=&ds;
    model &var = / dist=NEGBIN;
  run; quit;

  /* Pull Intercept and Alpha */
  proc sql noprint;
    select estimate, stderr into :b0, :seb0
    from nb_parms where upcase(parameter)='INTERCEPT';

    select estimate, stderr into :alpha, :sealpha
    from nb_parms where upcase(parameter) like '%ALPHA%';

    select count(*) into :n from &ds;
  quit;

  %let mu_hat = %sysfunc(exp(&b0));
  %let r_hat  = %sysevalf(1/&alpha);
  %let p_hat  = %sysevalf(&r_hat/(&r_hat + &mu_hat));

  %put NOTE: Baseline NB -> mu=&mu_hat r=&r_hat p=&p_hat (alpha=&alpha)  n=&n;

  /*---- 1) Build bootstrap sample set ----*/
  %if &__method = %str() %then %let __method=&_method;
  %if &__method = NP %then %do;  /* Nonparametric bootstrap: resample original */
    proc surveyselect data=&ds
      out=boot_all
      method=urs samprate=1 outhits reps=&B seed=&seed;
      id &var;
    run;
    proc sort data=boot_all; by replicate; run;
    data boot_all; set boot_all(rename=(&var=y)); keep replicate y; run;
  %end;
  %else %if &__method = PAR %then %do;  /* Parametric bootstrap: simulate NB */
    data boot_all;
      call streaminit(&seed);
      do replicate=1 to &B;
        do i=1 to &n;
          y = rand('NEGBINOMIAL', &p_hat, &r_hat);
          output;
        end;
      end;
      drop i;
    run;
  %end;
  %else %do;
    %put ERROR: METHOD=&method not recognized. Use METHOD=NP or METHOD=PAR.;
    %return;
  %end;

  /*---- 2) Refit NB by replicate ----*/
  ods exclude all;
  proc countreg data=boot_all;
    by replicate;
    model y = / dist=NEGBIN;
    ods output ParameterEstimates=_parms_;
  run; quit;
  ods select all;

  /* Keep only Intercept and Alpha rows; pivot to wide per replicate */
  proc sort data=_parms_; by replicate parameter; run;
  data _parms_keep; set _parms_;
    where upcase(parameter)='INTERCEPT' or upcase(parameter) like '%ALPHA%';
    length par $9;
    par = upcase(parameter);
    keep replicate par estimate;
  run;

  proc transpose data=_parms_keep out=_parms_wide prefix=EST_;
    by replicate;
    id par;
    var estimate;
  run;

  /* Some replicates may fail to converge; drop those with missing alpha or intercept */
  data NB_Boot_Estimates;
    set _parms_wide;
    if missing(EST_INTERCEPT) or (missing(EST__ALPHA) and missing(EST_ALPHA)) then delete;
    /* Normalize alpha column name */
    alpha = coalesce(EST__ALPHA, EST_ALPHA);
    b0    = EST_INTERCEPT;

    /* Derived parameters */
    mu = exp(b0);
    r  = 1/alpha;
    p  = r/(r+mu);

    keep replicate b0 alpha mu r p;
  run;

  /*---- 3) Percentile CIs & SEs from bootstrap ----*/
  proc means data=NB_Boot_Estimates noprint;
    var mu r p alpha b0;
    output out=_boot_se mean= / autoname;
  run;

  proc univariate data=NB_Boot_Estimates noprint;
    var mu r p alpha b0;
    output out=NB_Boot_CIs pctlpre=mu_ r_ p_ alpha_ b0_ pctlpts=2.5 97.5;
  run;

  /* Assemble summary with baseline point estimates and bootstrap dispersion */
  data NB_Boot_Summary;
    length Param $12;
    set NB_Boot_CIs;
    /* Pull SEs */
    if _n_=1 then set _boot_se(rename=(
      mu_Mean   = mu_bar
      r_Mean    = r_bar
      p_Mean    = p_bar
      alpha_Mean= alpha_bar
      b0_Mean   = b0_bar
    ));
    /* Baseline point estimates */
    mu_hat   = &mu_hat; r_hat=&r_hat; p_hat=&p_hat; alpha_hat=&alpha; b0_hat=&b0;

    /* Relative CI width helper macro-like logic via data step */
    Param='mu (mean)';  Estimate=mu_hat;  LCL=mu_2_5;   UCL=mu_97_5;   RCIW=(UCL-LCL)/max(abs(Estimate),1e-12); output;
    Param='alpha';      Estimate=alpha_hat;LCL=alpha_2_5;UCL=alpha_97_5;RCIW=(UCL-LCL)/max(abs(Estimate),1e-12); output;
    Param='r=1/alpha';  Estimate=r_hat;   LCL=r_2_5;    UCL=r_97_5;    RCIW=(UCL-LCL)/max(abs(Estimate),1e-12); output;
    Param='p';          Estimate=p_hat;   LCL=p_2_5;    UCL=p_97_5;    RCIW=(UCL-LCL)/max(abs(Estimate),1e-12); output;
    Param='b0 (log mu)';Estimate=b0_hat;  LCL=b0_2_5;   UCL=b0_97_5;   RCIW=(UCL-LCL)/max(abs(Estimate),1e-12); output;

    drop mu_: r_: p_: alpha_: b0_: mu_bar r_bar p_bar alpha_bar b0_bar
         mu_hat r_hat p_hat alpha_hat b0_hat;
  run;

  title "Bootstrap Stability — &__method method (&B reps) for &ds (&var)";
  proc print data=NB_Boot_Summary noobs label;
    label Estimate='Point' LCL='Boot 2.5%' UCL='Boot 97.5%' RCIW='Rel. CI Width';
  run;
  title;

%mend;

/* ==== EXAMPLES ==== */
/* Nonparametric bootstrap (resample original) */
%nb_bootstrap_stability(ds=work.frequency_et2, var=frequency, B=1000, seed=20241113, method=NP);

/* Parametric bootstrap (simulate from fitted NB) */
/* %nb_bootstrap_stability(ds=work.frequency_et2, var=frequency, B=1000, seed=20241113, method=PAR); */
