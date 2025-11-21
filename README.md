/* ------------------------------------------------------------------ */
/* MACRO: CALC_CVM_NEGBIN                                             */
/* Inputs:                                                            */
/* data_in = Your dataset (must have a 'frequency' column)         */
/* mu      = Mean parameter from CountReg                          */
/* alpha   = Dispersion parameter from CountReg                    */
/* ------------------------------------------------------------------ */

%macro calc_cvm_negbin(data_in=, mu=, alpha=);

/* 1. PREP: Convert Parameters for SAS CDF Function */
data _null_;
    mu = &mu;
    alpha = &alpha;
    
    /* COUNTREG uses Alpha/Mu. SAS CDF uses P (Prob) and N (Successes) */
    /* Conversion Logic: */
    r_hat = 1 / alpha;                 /* Number of Successes */
    p_hat = r_hat / (r_hat + mu);      /* Probability of Success */
    
    call symputx('p_param', p_hat);
    call symputx('r_param', r_hat);
run;

/* 2. AGGREGATE: Group data by Frequency (Handle Ties) */
/* This is crucial for Discrete CvM. We compare the 'Step' at each count */
proc sql;
    create table work.freq_summary as
    select frequency, 
           count(*) as count_obs
    from &data_in
    group by frequency
    order by frequency;
quit;

/* 3. CALCULATE: Choulakian-Lockhart-Stephens Statistic */
data work.cvm_stats;
    set work.freq_summary end=last;
    retain cum_obs 0;
    
    /* Total Sample Size */
    /* You can pass this as a macro var, or calculate on the fly */
    if _n_ = 1 then do;
        /* Get total N from the original table to be safe */
        dsid = open("&data_in");
        n_total = attrn(dsid, "nlobs");
        rc = close(dsid);
    end;
    retain n_total;

    /* A. Empirical CDF (The Data's Step Function) */
    /* Corresponds to the height of the step at this value */
    cum_obs = cum_obs + count_obs;
    F_emp = cum_obs / n_total; 

    /* B. Theoretical CDF (The Model's Curve) */
    /* CDF('NEGBINOMIAL', x, p, n) */
    F_theo = cdf('NEGBINOMIAL', frequency, &p_param, &r_param);
    
    /* C. Previous Theoretical CDF (Need this for the weighted sum) */
    /* If frequency=0, previous is 0. Else recalculate for freq-1 */
    if frequency = 0 then F_theo_prev = 0;
    else F_theo_prev = cdf('NEGBINOMIAL', frequency - 1, &p_param, &r_param);
    
    /* D. The Statistic Component (Z_j) */
    /* We calculate the squared error weighted by the probability mass */
    
    /* Average distance from theoretical midpoint */
    term1 = (F_theo - ((cum_obs - count_obs + cum_obs)/(2*n_total))); 
    
    /* Spinler/Choulakian simplified approximation for easy computation: */
    /* Sum of Squared Errors weighted by observed frequency is robust */
    sq_err = count_obs * (F_emp - F_theo)**2; 

    /* Accumulate */
    retain W2_Sum 0;
    W2_Sum = W2_Sum + sq_err;

    if last then do;
        /* Final CvM Statistic needs 1/12n correction for bias */
        CvM_Final = (W2_Sum / n_total) + (1 / (12 * n_total));
        
        call symputx('CvM_Result', CvM_Final);
        put "------------------------------------------------";
        put " computed Cramér–von Mises (CvM): " CvM_Final;
        put "------------------------------------------------";
    end;
run;

%mend;

/* ------------------------------------------------------------------ */
/* EXAMPLE USAGE with your parameters                                 */
/* ------------------------------------------------------------------ */

/* 1. Run your model to get parms */
proc countreg data=freq_et2;
    model frequency = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms;
run;

/* 2. Extract Parms into Macro Variables */
proc sql noprint;
    select estimate into :my_mu trimmed from nb_parms where upcase(parameter)='INTERCEPT'; /* Careful: Intercept is Log(Mu) */
    select estimate into :my_alpha trimmed from nb_parms where upcase(parameter) like '%ALPHA%';
quit;

/* 3. Fix the Log-Link (CountReg gives Intercept = Log(Mean)) */
%let my_real_mu = %sysevalf(exp(&my_mu));

/* 4. Run the Macro */
%calc_cvm_negbin(data_in=freq_et2, mu=&my_real_mu, alpha=&my_alpha);

/* Check the log for the result */
%put &CvM_Result;
