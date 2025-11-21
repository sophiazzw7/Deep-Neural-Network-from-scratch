/*===============================================
  ET7 severity mixture model + CvM statistic
===============================================*/
libname ops "/sasdata/mrmg2/users/G07267/MOD13638_2025/Output";

/* 1. ET7 data, truncated at 10,000 */
data severity_et7_10k;
    set ops.cleaned_severity_data;
    if strip('Basel Event Type Level 1'n) =
       "ET7 - Execution Delivery and Process Management"
       and gross_loss >= 10000;
run;

%let L_trunc = 10000;

/* 2. Mixture parameters (same as your quantile code) */
%let intercept_truncexp = 9.4179;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp.)));

%let intercept_lognorm  = 11.6984;
%let scale_lognorm      = 1.6015;
%let sdlog_lognorm      = %sysevalf(%sysfunc(sqrt(&scale_lognorm.)));

%let mix_p1 = 0.7036;
%let mix_p2 = 0.2964;

/* 3. Sample size */
proc sql noprint;
    select count(*) into :N_obs trimmed
    from severity_et7_10k;
quit;
%put NOTE: N_obs=&N_obs.;

/* 4. Sort data for CvM calculation */
proc sort data=severity_et7_10k;
    by gross_loss;
run;

/* 5. Compute CvM terms: sum_i (F(x_i) - (2i-1)/(2n))^2 */
data cvm_terms;
    set severity_et7_10k;
    by gross_loss;

    retain i;
    if _N_ = 1 then i = 0;
    i + 1;

    /* ----- mixture CDF at x = gross_loss ----- */
    x = gross_loss;

    if x < &L_trunc. then do;
        F_exp  = 0;
        F_logn = 0;
    end;
    else do;
        /* truncated exponential: L + Exp(sigma) */
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);

        /* truncated lognormal at L_trunc */
        FL = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0 = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);

        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;

    F_model = &mix_p1. * F_exp + &mix_p2. * F_logn;

    /* empirical mid-ranks (2i−1)/2n */
    F_emp_mid = (2*i - 1) / (2*&N_obs.);

    diff  = F_model - F_emp_mid;
    diff2 = diff*diff;
run;

/* 6. Sum and compute CvM = 1/(12n) + Σ diff^2 */
proc means data=cvm_terms noprint;
    var diff2;
    output out=cvm_sum sum = sum_diff2;
run;

data cvm_stat;
    set cvm_sum;
    N   = &N_obs.;
    CvM = (1/(12*N)) + sum_diff2;
run;

proc print data=cvm_stat;
    title "Cramer-von Mises Statistic for ET7 Severity Mixture";
run;
