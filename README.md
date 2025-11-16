/* Reuse fitted params */
%put NOTE: b0=&b0  alpha=&alpha  -> mu=&mu_hat r=&r_hat p=&p_hat;

/* Count of obs (n) and dataset name to sample from */
%let ds_in = frequency_et2;
proc sql noprint; select count(*) into :n from &ds_in; quit;

%macro nb_boot(B=1000, seed=20241113);
  /* Container for bootstrap estimates */
  data NB_BOOT; length iter 8 mu r p alpha b0 8.; stop; run;

  %do iter=1 %to &B;
    /* Simulate counts from fitted NB (same n as data) */
    data boot;
      call streaminit(&seed + &iter);
      do i=1 to &n;
        y = rand('NEGBINOMIAL', &p_hat, &r_hat);
        output;
      end;
    run;

    /* Refit NB to the bootstrap sample */
    ods exclude all;
    proc countreg data=boot;
      model y = / dist=NEGBIN;
      ods output ParameterEstimates=_parms_;
    quit;
    ods select all;

    /* Pull params; if fit fails, skip iteration */
    %local b0_b a_b;
    %let b0_b=.; %let a_b=.;

    proc sql noprint;
      select estimate into :b0_b from _parms_ where upcase(parameter)='INTERCEPT';
      select estimate into :a_b  from _parms_ where upcase(parameter) like '%ALPHA%';
    quit;

    %if %sysevalf(%superq(b0_b)=,boolean) or %sysevalf(%superq(a_b)=,boolean) %then %do; %end;
    %else %do;
      %let mu_b = %sysfunc(exp(&b0_b));
      %let r_b  = %sysevalf(1/&a_b);
      %let p_b  = %sysevalf(&r_b/(&r_b + &mu_b));
      data NB_BOOT; set NB_BOOT end=eof;
        output;
      run;
      data NB_BOOT; set NB_BOOT end=eof;
        if _n_=1 then do; iter=&iter; mu=&mu_b; r=&r_b; p=&p_b; alpha=&a_b; b0=&b0_b; output; end;
      run;
    %end;
  %end;

  /* Percentile CIs (2.5%, 97.5%) */
  proc univariate data=NB_BOOT noprint;
    var mu r p alpha b0;
    output out=NB_Boot_CIs pctlpre=mu_ r_ p_ alpha_ b0_ pctlpts=2.5 97.5;
  run;

  /* Assemble a neat table with point estimates + bootstrap CIs */
  data NB_Boot_Summary;
    length Param $12;
    set NB_Boot_CIs;
    Param='mu (mean)';  Estimate=&mu_hat;  LCL=mu_2_5;   UCL=mu_97_5;   output;
    Param='alpha';      Estimate=&alpha;   LCL=alpha_2_5;UCL=alpha_97_5;output;
    Param='r=1/alpha';  Estimate=&r_hat;   LCL=r_2_5;    UCL=r_97_5;    output;
    Param='p';          Estimate=&p_hat;   LCL=p_2_5;    UCL=p_97_5;    output;
  run;

%mend;

%nb_boot(B=1000);

/* Show bootstrap CIs */
proc print data=NB_Boot_Summary noobs label;
  label Estimate='Point' LCL='Boot 2.5%' UCL='Boot 97.5%';
run;
