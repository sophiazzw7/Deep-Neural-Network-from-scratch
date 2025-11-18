/* Anderson-Darling Test for QUARTERLY Operational Risk Frequency */

/* Step 1: Check the quarterly data */
proc print data=ops.ef_dataset(obs=10);
    var data_date event_type frequency;
    title "First 10 Quarterly Observations";
run;

proc means data=ops.ef_dataset n mean var std cv;
    var frequency;
    title "Quarterly Frequency Statistics";
run;

/* Step 2: Test for time trend (important for time series data) */
data trend_check;
    set ops.ef_dataset;
    time_index = _N_;  /* Sequential time index */
run;

proc reg data=trend_check;
    model frequency = time_index;
    title "Trend Test for Quarterly Frequency";
    title2 "If p-value < 0.05, significant trend exists";
run;
quit;

/* Step 3: Plot time series */
proc sgplot data=ops.ef_dataset;
    series x=data_date y=frequency / markers;
    title "Quarterly Frequency Over Time";
    xaxis label="Quarter";
    yaxis label="Frequency Count";
run;

/* Step 4: Fit Negative Binomial to QUARTERLY frequencies */
proc countreg data=ops.ef_dataset;
    model frequency = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms;
    title "Negative Binomial Fit - Quarterly Frequency";
run;

/* Step 5: Extract parameters */
proc sql noprint;
    select estimate into :b0 trimmed
    from nb_parms
    where upcase(Parameter) = 'INTERCEPT';
    
    select estimate into :alpha trimmed
    from nb_parms
    where upcase(Parameter) in ('DISPERSION', '_ALPHA', 'ALPHA');
quit;

/* Step 6: Calculate converted parameters */
%let mu_hat = %sysfunc(exp(&b0));
%let r_hat = %sysevalf(1/(&alpha));
%let p_hat = %sysevalf(&r_hat / (&r_hat + &mu_hat));

%put ===== QUARTERLY MODEL PARAMETERS =====;
%put Intercept (b0) = &b0;
%put Dispersion (alpha) = &alpha;
%put Mean per quarter (mu) = &mu_hat;
%put Shape (r) = &r_hat;
%put Success prob (p) = &p_hat;
%put Expected annual = %sysevalf(&mu_hat * 4);
%put ====================================;

/* Step 7: Anderson-Darling Test */
proc iml;

start AD_NB(x, p, r);
    call sort(x, 1);
    n = nrow(x);
    F = j(n,1,.);
    
    do i = 1 to n;
        F[i] = cdf("NEGBINOMIAL", floor(x[i]), p, r);
    end;
    
    do i = 1 to n;
        if F[i] < 1e-10 then F[i] = 1e-10;
        if F[i] > 1-1e-10 then F[i] = 1-1e-10;
    end;
    
    idx = T(1:n);
    A2 = -n - (1/n) * sum((2*idx-1)#(log(F) + log(1 - F[n+1-idx])));
    
    return(A2);
finish;

use ops.ef_dataset;
    read all var {frequency} into y;
close;

n = nrow(y);
p = &p_hat;
r = &r_hat;
mu = &mu_hat;

print "===== ANDERSON-DARLING TEST =====";
print "Testing: Quarterly frequency counts";
print "Sample size (quarters):" n;
print "Parameters: p=" p "r=" r "mu=" mu;

A2_obs = AD_NB(y, p, r);
print "Observed A2 statistic:" A2_obs;

/* Bootstrap */
nBoot = 10000;
call randseed(12345);
A2_boot = j(nBoot,1,.);

print "Running bootstrap with" nBoot "replications...";

do b = 1 to nBoot;
    yb = j(n,1,.);
    call randgen(yb, "NEGBINOMIAL", p, r);
    A2_boot[b] = AD_NB(yb, p, r);
end;

pval = (1 + sum(A2_boot >= A2_obs)) / (nBoot + 1);

print " ";
print "===== RESULTS =====";
print "Observed A2:" A2_obs;
print "Bootstrap p-value:" pval;
print " ";

if pval < 0.05 then do;
    print "Decision: REJECT H0 (p < 0.05)";
    print "Conclusion: Negative Binomial does NOT fit quarterly frequency well";
    print "Consider: Alternative distributions or detrending";
end;
else do;
    print "Decision: FAIL TO REJECT H0 (p >= 0.05)";
    print "Conclusion: Negative Binomial fits quarterly frequency adequately";
end;

print "===================";

/* Save results */
results = A2_obs || pval || p || r || mu || nBoot || n;
create AD_NB_Result from results
    [colname={"A2_obs" "pval" "p_hat" "r_hat" "mu_hat" "nBoot" "n_obs"}];
append from results;
close AD_NB_Result;

create AD_NB_Bootstrap from A2_boot[colname={"A2_boot"}];
append from A2_boot;
close AD_NB_Bootstrap;

quit;

/* Step 8: Print results */
proc print data=AD_NB_Result noobs;
    title "Anderson-Darling Test Results - Quarterly Frequency";
    format pval 6.4 A2_obs p_hat r_hat mu_hat 10.4;
run;

/* Step 9: Visualizations */
proc sgplot data=AD_NB_Bootstrap;
    histogram A2_boot / nbins=50 fillattrs=(color=lightblue);
    refline &A2_obs / axis=x lineattrs=(color=red thickness=2) 
            label="Observed A2";
    title "Bootstrap Distribution of Anderson-Darling Statistic";
    title2 "Quarterly Frequency Model";
    xaxis label="A2 Statistic";
    yaxis label="Frequency";
run;

/* Step 10: Important Notes */
data _null_;
    file print;
    put "===== IMPORTANT INTERPRETATION NOTES =====";
    put " ";
    put "1. TIME SERIES CONSIDERATIONS:";
    put "   - Your data are quarterly time series (not independent)";
    put "   - Check the trend test results above";
    put "   - If significant trend exists, consider detrending first";
    put " ";
    put "2. INDEPENDENCE ASSUMPTION:";
    put "   - AD test assumes independent observations";
    put "   - Quarterly data may have autocorrelation";
    put "   - Test results should be interpreted cautiously";
    put " ";
    put "3. QUARTERLY vs ANNUAL:";
    put "   - You are modeling QUARTERLY frequency (~&mu_hat events/quarter)";
    put "   - Expected ANNUAL frequency = &mu_hat x 4 = %sysevalf(&mu_hat * 4)";
    put "   - Make sure this matches your MDD requirements";
    put " ";
    put "4. ALTERNATIVE APPROACHES:";
    put "   - Consider Poisson regression with trend/seasonality";
    put "   - Use rolling window if parameters change over time";
    put "   - Check for structural breaks in the time series";
    put " ";
    put "==========================================";
run;
