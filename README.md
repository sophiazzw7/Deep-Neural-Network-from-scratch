/* Baseline NB fit on original data */
ods output ParameterEstimates=nb_parms;
proc countreg data=frequency_et2;
  model frequency = / dist=NEGBIN;
run;
quit;

/* Pull intercept and alpha from COUNTREG */
proc sql noprint;
  select estimate into :b0    from nb_parms where upcase(parameter)='INTERCEPT';
  select estimate into :alpha from nb_parms where upcase(parameter) like '%ALPHA%';
quit;

/* Point estimates in NB parameterization */
%let mu_hat = %sysfunc(exp(&b0));                 /* mean */
%let r_hat  = %sysevalf(1/&alpha);                /* dispersion (size) */
%let p_hat  = %sysevalf(&r_hat/(&r_hat + &mu_hat));   /* success prob */

%put NOTE: Baseline NB fit: b0=&b0 alpha=&alpha mu_hat=&mu_hat r_hat=&r_hat p_hat=&p_hat;
%macro nb_boot(ds_in=frequency_et2, var=frequency, B=1000, seed=20241115);

  /* Count of observations (we’ll simulate same n each time) */
  proc sql noprint;
    select count(*) into :n from &ds_in;
  quit;

  /* Container for bootstrap estimates */
  data NB_BOOT;
    length iter 8 mu r p alpha b0 8.;
    stop;
  run;

  %do iter = 1 %to &B;

    /*---- Simulate a bootstrap sample from fitted NB ----*/
    data boot;
      call streaminit(&seed + &iter);
      do i = 1 to &n;
        y = rand('NEGBINOMIAL', &p_hat, &r_hat);   /* parametric bootstrap */
        output;
      end;
      drop i;
    run;

    /*---- Refit NB to the bootstrap sample ----*/
    ods exclude all;
    ods output ParameterEstimates=_parms_;
    proc countreg data=boot;
      model y = / dist=NEGBIN;
    run;
    quit;
    ods select all;

    /* Pull intercept and alpha from this refit */
    %local b0_b a_b mu_b r_b p_b;
    %let b0_b=;
    %let a_b=;

    proc sql noprint;
      select estimate into :b0_b from _parms_ where upcase(parameter)='INTERCEPT';
      select estimate into :a_b  from _parms_ where upcase(parameter) like '%ALPHA%';
    quit;

    /* If refit failed or alpha invalid, skip this iteration */
    %if %sysevalf(%superq(b0_b)=,boolean) or %sysevalf(%superq(a_b)=,boolean) %then %do;
      %put NOTE: Iter &iter skipped (no NB params returned).;
    %end;
    %else %do;
      %let mu_b = %sysfunc(exp(&b0_b));
      %let r_b  = %sysevalf(1/&a_b);

      %if %sysevalf(&r_b <= 0) %then %do;
        %put NOTE: Iter &iter skipped (nonpositive r_b=&r_b).;
      %end;
      %else %do;
        %let p_b = %sysevalf(&r_b/(&r_b + &mu_b));

        /* Append one row with this bootstrap estimate */
        data _one;
          iter = &iter;
          mu   = &mu_b;
          r    = &r_b;
          p    = &p_b;
          alpha= &a_b;
          b0   = &b0_b;
        run;

        proc append base=NB_BOOT data=_one force; run;
      %end;
    %end;

  %end;  /* end iter loop */

  /*---- If we have any successful refits, build percentile CIs ----*/
  %let dsid = %sysfunc(open(NB_BOOT));
  %let nboot = %sysfunc(attrn(&dsid, nlobs));
  %let rc = %sysfunc(close(&dsid));

  %if &nboot > 0 %then %do;
    %put NOTE: &nboot successful bootstrap refits out of &B attempts.;

    proc univariate data=NB_BOOT noprint;
      var mu r p alpha b0;
      output out=NB_Boot_CIs
        pctlpre=mu_ r_ p_ alpha_ b0_
        pctlpts=2.5 97.5;
    run;

    data NB_Boot_Summary;
      length Param $12;
      set NB_Boot_CIs;

      Param='mu (mean)';  Estimate=&mu_hat; LCL=mu_2_5;    UCL=mu_97_5;    output;
      Param='alpha';      Estimate=&alpha;  LCL=alpha_2_5; UCL=alpha_97_5; output;
      Param='r=1/alpha';  Estimate=&r_hat;  LCL=r_2_5;     UCL=r_97_5;     output;
      Param='p';          Estimate=&p_hat;  LCL=p_2_5;     UCL=p_97_5;     output;

      keep Param Estimate LCL UCL;
    run;

    title "Bootstrap Confidence Intervals for NB Parameters (&nboot successful)";
    proc print data=NB_Boot_Summary noobs label;
      label Estimate='Point' LCL='Boot 2.5%' UCL='Boot 97.5%';
    run; title;

  %end;
  %else %do;
    %put ERROR: NB_BOOT has 0 rows – all bootstrap refits failed. Check model or sample size.;
  %end;

%mend nb_boot;

/* Run the bootstrap */
%nb_boot(ds_in=frequency_et2, var=frequency, B=1000, seed=20241115);
