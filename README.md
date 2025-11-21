/* ------------------------------------------------------------------ */
/* STEP 1: STORE YOUR OBSERVED STATISTIC                              */
/* ------------------------------------------------------------------ */
/* This grabs the '1.50' you just calculated from your previous step  */
proc sql noprint;
    select W2_obs into :W2_observed trimmed from Obs_CvM;
quit;

%put NOTE: The Observed CvM Statistic is &W2_observed;

/* ------------------------------------------------------------------ */
/* STEP 2: THE BOOTSTRAP LOOP MACRO                                   */
/* This simulates data, refits the model, and calculates W2 1000x     */
/* ------------------------------------------------------------------ */

%macro Bootstrap_CvM(nSim=1000);

    /* Create an empty table to store results */
    data Boot_Results; stop; run;

    %do b = 1 %to &nSim;

        /* A. SIMULATE DATA based on your fitted parameters */
        /* We generate N_obs data points using your mu_hat and alpha */
        data sim_freq;
            call streaminit(&b + 12345); /* Random seed changes per loop */
            do i = 1 to &N_obs;
                /* Generate Negative Binomial Random Variate */
                /* Note: RAND('NEGBINOMIAL') uses P (success prob) and N (successes) */
                /* Using the variables you already calculated: &p_hat and &r_hat */
                frequency = rand('NEGBINOMIAL', &p_hat, &r_hat);
                output;
            end;
        run;

        /* B. REFIT THE MODEL to the simulated data */
        /* We must re-estimate parameters to capture 'Estimation Bias' */
        proc countreg data=sim_freq noprint;
            model frequency = / dist=NEGBIN;
            ods output ParameterEstimates = sim_parms;
        run;

        /* Extract Simulated Parameters */
        proc sql noprint;
            select estimate into :b0_sim trimmed from sim_parms where upcase(parameter)='INTERCEPT';
            select estimate into :alpha_sim trimmed from sim_parms where upcase(parameter) like '%ALPHA%';
        quit;

        /* Convert Parameters for CDF calculation */
        %let mu_sim = %sysfunc(exp(&b0_sim));
        %let r_sim  = %sysevalf(1/&alpha_sim);
        %let p_sim  = %sysevalf(&r_sim / (&r_sim + &mu_sim));

        /* C. CALCULATE CvM STATISTIC FOR SIMULATED DATA */
        /* Sort is required for the formula */
        proc sort data=sim_freq; by frequency; run;

        data sim_calc;
            set sim_freq end=last;
            retain sum_sq 0;
            
            /* Rank i */
            i = _N_;
            n = &N_obs;
            
            /* Theoretical CDF using Refitted Sim Params */
            F_mod = cdf('NEGBINOMIAL', frequency, &p_sim, &r_sim);
            
            /* Empirical CDF Midpoint */
            emp_mid = (i - 0.5) / n;
            
            /* Squared Difference */
            diff_sq = (F_mod - emp_mid)**2;
            sum_sq = sum_sq + diff_sq;
            
            if last then do;
                W2_Sim = sum_sq + (1 / (12 * n));
                output;
            end;
            keep W2_Sim;
        run;

        /* Append result to master table */
        proc append base=Boot_Results data=sim_calc force; run;

    %end;
%mend;

/* ------------------------------------------------------------------ */
/* STEP 3: EXECUTE AND CALCULATE P-VALUE                              */
/* ------------------------------------------------------------------ */

/* Run the loop (Be patient, this takes a minute) */
%Bootstrap_CvM(nSim=1000); 

/* Calculate P-Value */
data P_Value_Calc;
    set Boot_Results end=last;
    retain count_exceed 0;
    
    /* Count how many simulated stats are WORSE (higher) than observed */
    if W2_Sim >= &W2_observed then count_exceed = count_exceed + 1;
    
    if last then do;
        P_Value = count_exceed / _N_;
        
        /* Formatting for Report */
        format P_Value 6.3; 
        
        put "--------------------------------------------------";
        put " FINAL BOOTSTRAP RESULTS";
        put " Observed CvM Statistic: " &W2_observed;
        put " Bootstrap P-Value:      " P_Value;
        put "--------------------------------------------------";
    end;
run;

proc print data=P_Value_Calc; 
    var P_Value;
    title "CvM Goodness-of-Fit: Parametric Bootstrap P-Value";
run;
