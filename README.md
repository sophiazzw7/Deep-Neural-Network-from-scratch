/* Simplified Anderson-Darling Test for Annual Operational Risk Frequency */

/* Step 1: Aggregate quarterly data to annual totals */
proc sql;
    create table annual_frequency as
    select 
        year(data_date) as year,
        sum(frequency) as annual_freq
    from ops.ef_dataset
    where event_type = 'ET2 - External Fraud'
    group by calculated year
    order by year;
quit;

/* Step 2: Check the annual data */
proc print data=annual_frequency;
    title "Annual Frequency Data";
run;

proc means data=annual_frequency n mean var std;
    var annual_freq;
    title "Annual Frequency Statistics";
run;

/* Step 3: Fit Negative Binomial to annual frequencies */
proc countreg data=annual_frequency;
    model annual_freq = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms;
run;

/* Step 4: Extract parameters */
proc sql noprint;
    select estimate into :b0 trimmed
    from nb_parms
    where upcase(Parameter) = 'INTERCEPT';
    
    select estimate into :alpha trimmed
    from nb_parms
    where upcase(Parameter) in ('DISPERSION', '_ALPHA', 'ALPHA');
quit;

/* Step 5: Calculate converted parameters */
%let mu_hat = %sysfunc(exp(&b0));
%let r_hat = %sysevalf(1/(&alpha));
%let p_hat = %sysevalf(&r_hat / (&r_hat + &mu_hat));

%put Intercept b0 = &b0;
%put Dispersion alpha = &alpha;
%put Mean mu_hat = &mu_hat;
%put Shape r_hat = &r_hat;
%put Prob p_hat = &p_hat;

/* Step 6: Anderson-Darling Test */
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

use annual_frequency;
    read all var {annual_freq} into y;
close;

n = nrow(y);
p = &p_hat;
r = &r_hat;
mu = &mu_hat;

A2_obs = AD_NB(y, p, r);

print "Sample size:" n;
print "Observed A2 statistic:" A2_obs;

/* Bootstrap */
nBoot = 10000;
call randseed(12345);
A2_boot = j(nBoot,1,.);

do b = 1 to nBoot;
    yb = j(n,1,.);
    call randgen(yb, "NEGBINOMIAL", p, r);
    A2_boot[b] = AD_NB(yb, p, r);
end;

pval = (1 + sum(A2_boot >= A2_obs)) / (nBoot + 1);

print "Bootstrap p-value:" pval;

if pval < 0.05 then
    print "Result: REJECT null hypothesis - NB does not fit well";
else
    print "Result: FAIL TO REJECT - NB fits adequately";

/* Save results */
results = A2_obs || pval || p || r || mu || nBoot;
create AD_NB_Result from results
    [colname={"A2_obs" "pval" "p_hat" "r_hat" "mu_hat" "B"}];
append from results;
close AD_NB_Result;

create AD_NB_Bootstrap from A2_boot[colname={"A2_boot"}];
append from A2_boot;
close AD_NB_Bootstrap;

quit;

/* Step 7: Print results */
proc print data=AD_NB_Result noobs;
    title "Anderson-Darling Test Results";
    format pval 6.4 A2_obs p_hat r_hat mu_hat 10.4;
run;

/* Step 8: Plot bootstrap distribution */
proc sgplot data=AD_NB_Bootstrap;
    histogram A2_boot / nbins=50;
    refline &A2_obs / axis=x lineattrs=(color=red thickness=2) 
            label="Observed";
    title "Bootstrap Distribution of A2 Statistic";
    xaxis label="A2 Statistic";
    yaxis label="Frequency";
run;
