


/*---- Fit NB frequency model ----*/
proc countreg data=ops.edpm_dataset;
    model frequency = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms;
quit;

/* Extract NB parameters */
proc sql noprint;
    select estimate into :b0 trimmed
    from nb_parms
    where upcase(Parameter)='INTERCEPT';

    select estimate into :alpha trimmed
    from nb_parms
    where upcase(Parameter) like '%ALPHA%';
quit;

/* Convert to NB(mean, r) parameterization */
%let mu_hat = %sysfunc(exp(&b0));
%let r_hat  = %sysevalf(1/&alpha);
%let p_hat  = %sysevalf(&r_hat/(&r_hat+&mu_hat));

/* Empirical percentiles for ET2 frequency */
proc univariate data=ops.edpm_dataset noprint;
    var frequency;
    output out=emp_freq 
        pctlpts = 50 75 90 95 99
        pctlpre = Q_;
run;

%let nSim = 100000;

data sim_freq;
    call streaminit(12345);
    do sim_id = 1 to &nSim;
        call randgen(freq_sim, 'NEGBINOMIAL', &p_hat, &r_hat);
        output;
    end;
run;

proc univariate data=sim_freq noprint;
    var freq_sim;
    output out=sim_q
        pctlpts = 50 75 90 95 99
        pctlpre = S_;
run;

/* percentile per simulation */
proc univariate data=sim_freq noprint;
    by sim_id;
    var freq_sim;
    output out=sim_each 
        pctlpts = 50 75 90 95 99
        pctlpre = P_;
run;

/* Create 95% envelope */
proc means data=sim_each noprint;
    var P_50 P_75 P_90 P_95 P_99;
    output out=sim_band
        p2_5  = P2_Q50  P2_Q75  P2_Q90  P2_Q95  P2_Q99
        p50   = P50_Q50 P50_Q75 P50_Q90 P50_Q95 P50_Q99
        p97_5 = P97_Q50 P97_Q75 P97_Q90 P97_Q95 P97_Q99;
run;

data freq_quantile_test;
    merge emp_freq sim_band;
    /* no BY statement since both contain one row */
run;

proc print data=freq_quantile_test;
run;
