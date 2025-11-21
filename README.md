%macro Bootstrap_CvM(nSim=1000);

    /* 1. Create an empty table to store results */
    data Boot_Results; length W2_Sim 8; stop; run;

    %let success_count = 0;

    %do b = 1 %to &nSim;

        /* --- A. SIMULATE DATA --- */
        data sim_freq;
            call streaminit(&b + 12345);
            do i = 1 to &N_obs;
                frequency = rand('NEGBINOMIAL', &p_hat, &r_hat);
                output;
            end;
        run;

        /* Clean up previous iteration's parameter file */
        proc datasets lib=work nolist; delete sim_parms; quit;

        /* --- B. REFIT THE MODEL (Wrapped in Try/Catch Logic) --- */
        /* Use 'ods results off' to speed up and 'printing' options to suppress output */
        ods exclude all;
        ods noresults;
        
        proc countreg data=sim_freq;
            model frequency = / dist=NEGBIN;
            ods output ParameterEstimates = sim_parms;
        run;
        
        ods results on;
        ods exclude none;

        /* --- C. CHECK IF MODEL CONVERGED --- */
        /* Only proceed if sim_parms was actually created */
        %if %sysfunc(exist(sim_parms)) %then %do;

            /* Extract Simulated Parameters */
            proc sql noprint;
                select estimate into :b0_sim trimmed from sim_parms where upcase(parameter)='INTERCEPT';
                select estimate into :alpha_sim trimmed from sim_parms where upcase(parameter) like '%ALPHA%';
            quit;

            /* Convert Parameters for CDF calculation */
            %let mu_sim = %sysfunc(exp(&b0_sim));
            %let r_sim  = %sysevalf(1/&alpha_sim);
            %let p_sim  = %sysevalf(&r_sim / (&r_sim + &mu_sim));

            /* --- D. CALCULATE CVM STATISTIC --- */
            proc sort data=sim_freq; by frequency; run;

            data sim_calc;
                set sim_freq end=last;
                retain sum_sq 0;
                
                i = _N_;
                n = &N_obs;
                
                /* Theoretical CDF using Refitted Sim Params */
                F_mod = cdf('NEGBINOMIAL', frequency, &p_sim, &r_sim);
                
                emp_mid = (i - 0.5) / n;
                diff_sq = (F_mod - emp_mid)**2;
                sum_sq = sum_sq + diff_sq;
                
                if last then do;
                    W2_Sim = sum_sq + (1 / (12 * n));
                    output;
                end;
                keep W2_Sim;
            run;

            /* Append result */
            proc append base=Boot_Results data=sim_calc force; run;
            
            /* Increment success counter */
            %let success_count = %eval(&success_count + 1);
            
        %end;
        %else %do;
            %put WARNING: Iteration &b failed to converge. Skipping.;
        %end;

    %end;
    
    %put NOTE: Bootstrap completed. Successful iterations: &success_count out of &nSim;

%mend;

/* --- RUN IT --- */
%Bootstrap_CvM(nSim=1000);

/* --- CALCULATE FINAL P-VALUE --- */
proc sql noprint;
    select W2_obs into :W2_observed trimmed from Obs_CvM;
quit;

data P_Value_Calc;
    set Boot_Results end=last;
    retain count_exceed 0;
    
    /* Count how many simulated stats are WORSE (higher) than observed */
    if W2_Sim >= &W2_observed then count_exceed = count_exceed + 1;
    
    if last then do;
        P_Value = count_exceed / _N_;
        format P_Value 6.3; 
        put "--------------------------------------------------";
        put " FINAL BOOTSTRAP RESULTS";
        put " Total Successful Simulations: " _N_;
        put " Observed CvM Statistic: " &W2_observed;
        put " Bootstrap P-Value:      " P_Value;
        put "--------------------------------------------------";
    end;
run;

proc print data=P_Value_Calc; run;
