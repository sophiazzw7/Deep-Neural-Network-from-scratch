/* Baseline fit (you already have these macro vars set: b0, alpha, mu_hat, r_hat, p_hat) */
/* If not, re-run COUNTREG or GENMOD once to get b0 and alpha, then define:
   %let mu_hat = %sysfunc(exp(&b0));
   %let r_hat  = %sysevalf(1/&alpha);
   %let p_hat  = %sysevalf(&r_hat/(&r_hat + &mu_hat));                                   */

%macro nb_boot_fix(ds_in=frequency_et2, var=frequency, B=1000, seed=20241113);

  /* Prepare container */
  data NB_BOOT; length iter 8 mu r p alpha b0 8.; stop; run;

  /* Count of rows to simulate per replicate (parametric bootstrap) */
  proc sql noprint; select count(*) into :n from &ds_in; quit;

  %do iter=1 %to &B;

    /* ---- Simulate from fitted NB (parametric bootstrap) ---- */
    data boot;
      call streaminit(&seed + &iter);
      do i=1 to &n;
        y = rand('NEGBINOMIAL', &p_hat, &r_hat);  /* p = &p_hat, r = &r_hat */
        output;
      end;
      drop i;
    run;

    /* Skip degenerate samples (all zeros) */
    proc sql noprint;
      select sum(y), count(*) into :sumy, :ny from boot;
    quit;
    %if %sysevalf(&sumy = 0) %then %do; %goto next_iter; %end;

    /* ---- Refit NB on bootstrap sample with GENMOD (more robust than COUNTREG here) ---- */
    ods exclude all;
    ods output ParameterEstimates=_parms_(keep=parameter estimate)
               ModelFit=_fit_;
    proc genmod data=boot;
      model y = / dist=negbin link=log;   /* intercept-only NB */
    run; quit;
    ods select all;

    /* Pull intercept (b0) and NB k (r) from GENMOD */
    /* In GENMOD, the NB parameter reported is k = r (variance = mu + mu^2 / k) */
    %local b0_b r_b mu_b p_b alpha_b;
    %let b0_b=.; %let r_b=.; %let mu_b=.; %let p_b=.; %let alpha_b=.;

    /* Intercept */
    proc sql noprint;
      select estimate into :b0_b from _parms_ where upcase(parameter)='INTERCEPT';
    quit;

    /* NB k (r) comes from ModelFit table row "Dispersion" for NB — safer approach: */
    /* Some SAS versions instead put k in a row named 'Alpha' or 'Scale'. Grab both, prefer k>0 */
    %local _k1 _k2;
    %let _k1=.; %let _k2=.;

    /* Try to pull k (r) directly from ModelFit if available */
    proc sql noprint;
      /* Many installs put NB k under 'Scale' with label 'NegBin' or directly under 'Alpha' row. */
      select estimate into :_k1 from _fit_ where upcase(descr) like '%NEG%BIN%' or upcase(descr) like '%SCALE%';
      select estimate into :_k2 from _parms_ where upcase(parameter) in ('ALPHA','_ALPHA','NB') ;
    quit;

    /* Choose a positive r */
    %if %sysevalf(%superq(_k1)=,boolean) %then %let r_b=&_k2;
    %else %let r_b=&_k1;

    /* Fallback: if still missing or nonpositive, try COUNTREG once */
    %if %sysevalf(%superq(r_b)=,boolean) or %sysevalf(&r_b<=0) %then %do;
      ods exclude all;
      ods output ParameterEstimates=_parms2(keep=parameter estimate);
      proc countreg data=boot;
        model y = / dist=negbin;
      run; quit;
      ods select all;
      proc sql noprint;
        select estimate into :b0_b from _parms2 where upcase(parameter)='INTERCEPT';
        select estimate into :alpha_b from _parms2 where upcase(parameter) like '%ALPHA%';
      quit;
      %if not %sysevalf(%superq(alpha_b)=,boolean) and %sysevalf(&alpha_b>0) %then %let r_b=%sysevalf(1/&alpha_b);
    %end;

    /* If we still didn't get a valid r or b0, skip this replicate */
    %if %sysevalf(%superq(b0_b)=,boolean) or %sysevalf(%superq(r_b)=,boolean) or %sysevalf(&r_b<=0) %then %goto next_iter;

    /* Derived parameters */
    %let mu_b = %sysfunc(exp(&b0_b));
    %let p_b  = %sysevalf(&r_b/(&r_b + &mu_b));
    %let alpha_b = %sysevalf(1/&r_b);

    /* Append one row */
    data _one;
      iter=&iter; mu=&mu_b; r=&r_b; p=&p_b; alpha=&alpha_b; b0=&b0_b;
    run;
    proc append base=NB_BOOT data=_one force; run;

    %next_iter:
  %end;

  /* If we got some successful refits, compute percentile CIs */
  %let dsid=%sysfunc(open(NB_BOOT));
  %let rows=%sysfunc(attrn(&dsid, nlobs)); %let rc=%sysfunc(close(&dsid));

  %if &rows > 0 %then %do;
    proc univariate data=NB_BOOT noprint;
      var mu r p alpha b0;
      output out=NB_Boot_CIs pctlpre=mu_ r_ p_ alpha_ b0_ pctlpts=2.5 97.5;
    run;

    data NB_Boot_Summary;
      length Param $12;
      set NB_Boot_CIs;
      Param='mu (mean)';  Estimate=&mu_hat;  LCL=mu_2_5;   UCL=mu_97_5;   output;
      Param='alpha';      Estimate=%sysevalf(1/&r_hat); LCL=alpha_2_5; UCL=alpha_97_5; output;
      Param='r=1/alpha';  Estimate=&r_hat;   LCL=r_2_5;    UCL=r_97_5;    output;
      Param='p';          Estimate=&p_hat;   LCL=p_2_5;    UCL=p_97_5;    output;
    run;

    title "Bootstrap CIs from successful refits (&rows of &B)";
    proc print data=NB_Boot_Summary noobs label;
      label Estimate='Point' LCL='Boot 2.5%' UCL='Boot 97.5%';
    run; title;
  %end;
  %else %do;
    %put ERROR: No successful bootstrap refits. Consider smaller B, or keep COUNTREG but raise MAXITER, or switch to nonparametric bootstrap.;
  %end;

%mend;

/* Run it */
%nb_boot_fix(ds_in=frequency_et2, var=frequency, B=1000, seed=20241113);
