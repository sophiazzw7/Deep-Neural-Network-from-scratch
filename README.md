/*=============================================================================
  CORRECTED ANDERSON-DARLING TEST FOR OPERATIONAL RISK FREQUENCY
  Purpose: Test if ANNUAL frequency follows Negative Binomial distribution
  
  Key Changes from Original:
  1. Aggregates quarterly data to annual totals
  2. Fits NB distribution to annual frequencies
  3. Runs AD test on annual data (not quarterly)
  4. Adds diagnostic plots and checks
=============================================================================*/

/* Step 1: Clean workspace */
proc datasets lib=work nolist;
    delete nb_parms annual_frequency AD_NB_Result AD_NB_Bootstrap;
quit;

/* Step 2: Aggregate quarterly data to ANNUAL frequency */
proc sql;
    create table annual_frequency as
    select 
        year(data_date) as year,
        sum(frequency) as annual_freq,
        sum(gross_loss) as annual_gross_loss,
        sum(net_loss) as annual_net_loss,
        count(*) as n_quarters  /* Verify 4 quarters per year */
    from ops.ef_dataset
    where event_type = 'ET2 - External Fraud'
    group by calculated year
    order by year;
quit;

/* Step 3: Review aggregated annual data */
proc print data=annual_frequency;
    title "Annual Frequency Data for Anderson-Darling Test";
    title2 "Each row = one year of fraud events";
run;

proc means data=annual_frequency n mean var std cv min max;
    var annual_freq;
    title "Summary Statistics: Annual Frequency";
run;

/* Step 4: Check for time trends (important!) */
proc sgplot data=annual_frequency;
    series x=year y=annual_freq / markers;
    title "Annual Frequency Over Time";
    title2 "Check for Trends or Structural Breaks";
    xaxis label="Year";
    yaxis label="Annual Frequency Count";
run;

/* Step 5: Test for linear trend */
proc reg data=annual_frequency;
    model annual_freq = year;
    title "Linear Trend Test";
    title2 "p-value < 0.05 indicates significant trend";
run;
quit;

/* Step 6: Fit Negative Binomial distribution to ANNUAL frequencies */
proc countreg data=annual_frequency;
    model annual_freq = / dist=NEGBIN;
    output out=countreg_frequency_pred_negbin pred=pred_negbin;
    ods output ParameterEstimates = nb_parms;
    title "Negative Binomial Distribution Fit - Annual Frequency";
run;

/* Step 7: Extract parameters with error checking */
proc sql noprint;
    /* Count parameters */
    select count(*) into :n_parms trimmed
    from nb_parms;
    
    /* Extract Intercept (b0) */
    select count(*) into :has_intercept trimmed
    from nb_parms
    where upcase(Parameter) = 'INTERCEPT';
    
    select estimate into :b0 trimmed
    from nb_parms
    where upcase(Parameter) = 'INTERCEPT';
    
    /* Extract Dispersion parameter (try multiple possible names) */
    select count(*) into :has_disp trimmed
    from nb_parms
    where upcase(Parameter) in ('DISPERSION', '_ALPHA', 'ALPHA', 'SCALE');
    
    select estimate into :alpha trimmed
    from nb_parms
    where upcase(Parameter) in ('DISPERSION', '_ALPHA', 'ALPHA', 'SCALE');
quit;

/* Step 8: Verify parameter extraction */
%put ===== PARAMETER EXTRACTION CHECK =====;
%put Number of parameters found: &n_parms;
%put Has Intercept: &has_intercept;
%put Has Dispersion: &has_disp;
%put Intercept (b0): &b0;
%put Dispersion (alpha): &alpha;
%put ======================================;

/* Step 9: Calculate converted parameters */
%let mu_hat = %sysfunc(exp(&b0));
%let r_hat = %sysevalf(1/(&alpha));
%let p_hat = %sysevalf(&r_hat / (&r_hat + &mu_hat));

/* Step 10: Display all parameters */
data _null_;
    b0 = &b0;
    alpha = &alpha;
    mu_hat = &mu_hat;
    r_hat = &r_hat;
    p_hat = &p_hat;
    
    /* Calculate theoretical variance */
    variance_theoretical = mu_hat + alpha * mu_hat**2;
    
    /* Verify conversion */
    mean_from_rp = r_hat * (1-p_hat) / p_hat;
    
    put "======= PARAMETER SUMMARY =======";
    put "COUNTREG Output:";
    put "  Intercept (b0) = " b0;
    put "  Dispersion (alpha) = " alpha;
    put " ";
    put "Converted Parameters:";
    put "  Mean (mu_hat) = " mu_hat;
    put "  Shape (r_hat) = " r_hat;
    put "  Success prob (p_hat) = " p_hat;
    put " ";
    put "Model Properties:";
    put "  Expected mean = " mu_hat;
    put "  Expected variance = " variance_theoretical;
    put "  Overdispersion ratio (Var/Mean) = " variance_theoretical @;
    put mu_hat @;
    put " = " variance_theoretical / mu_hat;
    put " ";
    put "Verification:";
    put "  Mean from r & p = " mean_from_rp " (should match mu_hat)";
    put "=================================";
run;

/* Step 11: Anderson-Darling Test Implementation */
proc iml;

    /* Define AD_NB function */
    start AD_NB(x, p, r);
        /* Sort data in ascending order - CRITICAL! */
        call sort(x, 1);
        
        n = nrow(x);
        F = j(n,1,.);
        
        /* Calculate CDF for each sorted observation */
        do i = 1 to n;
            /* Use floor() for discrete distribution */
            F[i] = cdf("NEGBINOMIAL", floor(x[i]), p, r);
        end;
        
        /* Boundary protection to prevent log(0) errors */
        do i = 1 to n;
            if F[i] < 1e-10 then F[i] = 1e-10;
            if F[i] > 1-1e-10 then F[i] = 1-1e-10;
        end;
        
        /* Create index vector for weighting */
        idx = T(1:n);
        
        /* Calculate Anderson-Darling statistic */
        A2 = -n - (1/n) * sum((2*idx-1)#(log(F) + log(1 - F[n+1-idx])));
        
        return(A2);
    finish;
    
    /* Read ANNUAL frequency data */
    use annual_frequency;
        read all var {annual_freq} into y;
    close;
    
    n = nrow(y);
    p = &p_hat;
    r = &r_hat;
    mu = &mu_hat;
    
    print "===== ANDERSON-DARLING TEST: ANNUAL FREQUENCY =====";
    print "Sample size (years):" n;
    print "Parameters:" p r mu;
    print " ";
    
    /* Calculate observed A² statistic */
    A2_obs = AD_NB(y, p, r);
    print "Observed A² statistic:" A2_obs;
    
    /* Bootstrap procedure */
    nBoot = 10000;
    call randseed(12345);
    
    A2_boot = j(nBoot,1,.);
    
    print " ";
    print "Running bootstrap with" nBoot "replications...";
    print "(This may take a moment)";
    
    do b = 1 to nBoot;
        /* Generate bootstrap sample from fitted NB distribution */
        yb = j(n,1,.);
        call randgen(yb, "NEGBINOMIAL", p, r);
        
        /* Calculate A² for bootstrap sample */
        A2_boot[b] = AD_NB(yb, p, r);
        
        /* Progress indicator every 2000 iterations */
        if mod(b, 2000) = 0 then 
            print "  Completed" b "of" nBoot "iterations";
    end;
    
    /* Calculate p-value */
    pval = (1 + sum(A2_boot >= A2_obs)) / (nBoot + 1);
    
    /* Calculate bootstrap quantiles */
    call qntl(boot_quantiles, A2_boot, {0.05 0.10 0.25 0.50 0.75 0.90 0.95});
    
    print " ";
    print "===== RESULTS =====";
    print "Observed A² statistic:" A2_obs;
    print "Bootstrap p-value:" pval;
    print " ";
    print "Bootstrap distribution quantiles:";
    print "  5%:" (boot_quantiles[1]);
    print " 10%:" (boot_quantiles[2]);
    print " 25%:" (boot_quantiles[3]);
    print " 50%:" (boot_quantiles[4]);
    print " 75%:" (boot_quantiles[5]);
    print " 90%:" (boot_quantiles[6]);
    print " 95%:" (boot_quantiles[7]);
    print " ";
    
    if pval < 0.05 then
        print "Conclusion: REJECT null hypothesis (p < 0.05)";
    else
        print "Conclusion: FAIL TO REJECT null hypothesis (p >= 0.05)";
    
    print "===================";
    
    /* Save results */
    results = A2_obs || pval || p || r || mu || nBoot;
    create AD_NB_Result from results
        [colname={"A2_obs" "pval" "p_hat" "r_hat" "mu_hat" "B"}];
    append from results;
    close AD_NB_Result;
    
    /* Save bootstrap distribution */
    create AD_NB_Bootstrap from A2_boot[colname={"A2_boot"}];
    append from A2_boot;
    close AD_NB_Bootstrap;

quit;

/* Step 12: Visualize bootstrap distribution */
data AD_NB_Bootstrap;
    set AD_NB_Bootstrap;
    obs_A2 = &A2_obs;  /* Add observed value for reference line */
run;

proc sgplot data=AD_NB_Bootstrap;
    histogram A2_boot / nbins=50 fillattrs=(color=lightblue);
    refline obs_A2 / axis=x lineattrs=(color=red thickness=2 pattern=solid)
            label="Observed A²" labelloc=inside labelpos=max;
    density A2_boot / lineattrs=(color=darkblue thickness=2);
    title "Bootstrap Distribution of Anderson-Darling Statistic";
    title2 "Annual Frequency - 10,000 Bootstrap Samples";
    xaxis label="A² Statistic";
    yaxis label="Density";
run;

/* Step 13: Q-Q Plot for visual goodness-of-fit check */
proc iml;
    use annual_frequency;
        read all var {annual_freq} into y;
    close;
    
    call sort(y, 1);
    n = nrow(y);
    
    p = &p_hat;
    r = &r_hat;
    
    /* Calculate empirical probabilities */
    emp_prob = (T(1:n) - 0.5) / n;
    
    /* Calculate theoretical quantiles */
    theo_quant = j(n,1,.);
    do i = 1 to n;
        /* Use quantile function for NB distribution */
        theo_quant[i] = quantile("NEGBINOMIAL", emp_prob[i], p, r);
    end;
    
    /* Create dataset for plotting */
    qq_data = y || theo_quant;
    create qq_plot from qq_data[colname={"Observed" "Theoretical"}];
    append from qq_data;
    close qq_plot;
quit;

proc sgplot data=qq_plot aspect=1;
    scatter x=Theoretical y=Observed / markerattrs=(size=10 symbol=circlefilled);
    lineparm x=0 y=0 slope=1 / lineattrs=(color=red thickness=2 pattern=dash)
             legendlabel="Perfect Fit";
    title "Q-Q Plot: Negative Binomial Fit";
    title2 "Points near line indicate good fit";
    xaxis label="Theoretical Quantiles (NB Distribution)";
    yaxis label="Observed Annual Frequencies";
run;

/* Step 14: Print final results table */
proc print data=AD_NB_Result label noobs;
    label 
        A2_obs = "A² Statistic (Observed)"
        pval = "Bootstrap p-value"
        p_hat = "NB Success Probability (p)"
        r_hat = "NB Shape Parameter (r)"
        mu_hat = "Expected Annual Frequency (μ)"
        B = "Bootstrap Replications";
    format pval 6.4 A2_obs p_hat r_hat mu_hat 10.4;
    title "Anderson-Darling Test Results - Annual Frequency";
run;

/* Step 15: Interpretation Guide */
data _null_;
    pval = input(symget('pval'), best.);
    
    put "======= INTERPRETATION GUIDE =======";
    put " ";
    put "Null Hypothesis (H0): Annual frequency follows NB(r,p) distribution";
    put "Alternative (H1): Annual frequency does NOT follow NB(r,p)";
    put " ";
    put "Decision Rule (α = 0.05):";
    put "  - If p-value < 0.05: Reject H0 (NB is a poor fit)";
    put "  - If p-value >= 0.05: Fail to reject H0 (NB is adequate)";
    put " ";
    put "Your p-value: " pval;
    
    if pval < 0.05 then do;
        put " ";
        put "RESULT: REJECT the null hypothesis";
        put " ";
        put "Interpretation:";
        put "  The negative binomial distribution does NOT adequately";
        put "  describe your annual operational risk frequency.";
        put " ";
        put "Recommendations:";
        put "  1. Try alternative distributions (Poisson, Zero-Inflated NB)";
        put "  2. Check for outliers or structural breaks";
        put "  3. Consider time-varying models if trend exists";
        put "  4. Review the Q-Q plot for pattern of misfit";
    end;
    else do;
        put " ";
        put "RESULT: FAIL TO REJECT the null hypothesis";
        put " ";
        put "Interpretation:";
        put "  The negative binomial distribution provides an adequate";
        put "  fit to your annual operational risk frequency.";
        put " ";
        put "Notes:";
        put "  1. 'Adequate' does not mean 'perfect' fit";
        put "  2. Review Q-Q plot for visual confirmation";
        put "  3. With small sample size, test has limited power";
        put "  4. Consider using this distribution for LDA modeling";
    end;
    
    put " ";
    put "IMPORTANT CAVEATS:";
    put "  - Small sample size reduces test power";
    put "  - If n < 10 years, interpret results cautiously";
    put "  - Supplement with Q-Q plots and expert judgment";
    put "  - Consider regulatory requirements";
    put "====================================";
run;

/* Step 16: Calculate actual vs fitted comparison */
proc sql;
    create table fit_comparison as
    select 
        a.year,
        a.annual_freq as observed,
        &mu_hat as fitted_mean,
        a.annual_freq - &mu_hat as residual,
        (a.annual_freq - &mu_hat) / sqrt(&mu_hat + &alpha * &mu_hat**2) as std_residual
    from annual_frequency as a
    order by year;
quit;

proc print data=fit_comparison;
    title "Observed vs Fitted Annual Frequencies";
    title2 "Check for systematic patterns in residuals";
    format observed fitted_mean 8.2 residual std_residual 8.3;
run;

proc sgplot data=fit_comparison;
    scatter x=year y=std_residual / markerattrs=(size=10);
    refline 0 / lineattrs=(color=black);
    refline -2 2 / lineattrs=(color=red pattern=dash);
    title "Standardized Residuals Over Time";
    title2 "Points outside [-2, 2] indicate potential outliers";
    xaxis label="Year";
    yaxis label="Standardized Residual";
run;
