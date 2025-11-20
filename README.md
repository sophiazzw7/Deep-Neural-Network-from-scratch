/*============================================================*/
/*  Binned Relative Error (BRE) goodness-of-fit test          */
/*  ET7 – mixture severity (trunc at 10,000)                  */
/*============================================================*/

libname ops "/sasdata/mrmg2/users/G07267/MOD13638_2025/Output";

/*------------------------------
  0. Prep ET7 severity data
------------------------------*/
data severity_et7_10k;
    set ops.cleaned_severity_data;
    if strip('Basel Event Type Level 1'n) = 
       "ET7 - Execution Delivery and Process Management"
       and gross_loss >= 10000;
run;

%let L_trunc = 10000;

/*------------------------------
  1. Mixture parameters
------------------------------*/
%let intercept_truncexp = 9.4179;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp.)));

%let intercept_lognorm  = 11.6984;
%let scale_lognorm      = 1.6015;
%let sdlog_lognorm      = %sysevalf(%sysfunc(sqrt(&scale_lognorm.)));

%let mix_p1 = 0.7036;   /* exponential weight  */
%let mix_p2 = 0.2964;   /* lognormal weight    */

/* Sample size */
proc sql noprint;
    select count(*) into :N_obs trimmed
    from severity_et7_10k;
quit;
%put NOTE: N_obs=&N_obs.;

/*------------------------------
  2. Empirical quantiles (bin cuts)
     [L_trunc,Q50), [Q50,Q75), [Q75,Q90),
     [Q90,Q95), [Q95,+inf)
------------------------------*/
proc univariate data=severity_et7_10k noprint;
    var gross_loss;
    output out=et7_q pctlpts=50 75 90 95 pctlpre=Q;
run;

data _null_;
    set et7_q;
    call symputx('Q50', Q50);
    call symputx('Q75', Q75);
    call symputx('Q90', Q90);
    call symputx('Q95', Q95);
run;

%put NOTE: Q50=&Q50. Q75=&Q75. Q90=&Q90. Q95=&Q95.;

/*-------------------------------------------------------
  Helper snippet: mixture CDF at value x (>= L_trunc)
  (implemented inline in each bin to avoid macro issues)
--------------------------------------------------------*/
/*  F_model = p1 * F_exp_trunc + p2 * F_logn_trunc     */
/*  where F_logn_trunc uses lognormal truncated at L.  */

/*------------------------------
  3. Model bin probabilities
------------------------------*/
data bin_probs;
    length bin 8;
    retain F_L F_U P_model;

    /* ---- Bin 1: [L_trunc, Q50) ---- */
    bin = 1;

    /* lower boundary */
    x = &L_trunc.;
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_L = F_model;

    /* upper boundary */
    x = &Q50.;
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_U = F_model;

    P_model = F_U - F_L;
    output;

    /* ---- Bin 2: [Q50, Q75) ---- */
    bin = 2;

    x = &Q50.;
    /* lower */
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_L = F_model;

    x = &Q75.;
    /* upper */
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_U = F_model;

    P_model = F_U - F_L;
    output;

    /* ---- Bin 3: [Q75, Q90) ---- */
    bin = 3;

    x = &Q75.;
    /* lower */
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_L = F_model;

    x = &Q90.;
    /* upper */
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_U = F_model;

    P_model = F_U - F_L;
    output;

    /* ---- Bin 4: [Q90, Q95) ---- */
    bin = 4;

    x = &Q90.;
    /* lower */
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_L = F_model;

    x = &Q95.;
    /* upper */
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_U = F_model;

    P_model = F_U - F_L;
    output;

    /* ---- Bin 5: [Q95, +inf) ---- */
    bin = 5;

    x = &Q95.;
    /* lower */
    if x < &L_trunc. then do;
        F_exp = 0; F_logn = 0;
    end;
    else do;
        F_exp = cdf('EXPONENTIAL', x - &L_trunc., &sigma_truncexp.);
        FL    = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0    = cdf('LOGNORMAL', x,          &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn = 0;
        else F_logn = (F0 - FL) / (1 - FL);
    end;
    F_model = &mix_p1.*F_exp + &mix_p2.*F_logn;
    F_L = F_model;

    /* upper is +inf => CDF = 1 */
    F_U = 1;
    P_model = F_U - F_L;
    output;

    drop x F_exp F_logn FL F0 F_model;
run;

/* Expected counts per bin */
data bin_probs;
    set bin_probs;
    E_count = &N_obs. * P_model;
run;

/*------------------------------
  4. Observed counts per bin
------------------------------*/
data severity_bins;
    set severity_et7_10k;
    length bin 8;
    if gross_loss <  &Q50. then bin = 1;
    else if gross_loss < &Q75. then bin = 2;
    else if gross_loss < &Q90. then bin = 3;
    else if gross_loss < &Q95. then bin = 4;
    else bin = 5;
run;

proc sql;
    create table bin_obs as
    select bin, count(*) as O_count
    from severity_bins
    group by bin;
quit;

/*------------------------------
  5. Merge & compute BRE metrics
------------------------------*/
data bin_fit;
    merge bin_probs bin_obs;
    by bin;
    if E_count > 0 then RelErr = (O_count - E_count) / E_count;
    AbsRelErr = abs(RelErr);
run;

proc print data=bin_fit;
    var bin O_count E_count RelErr AbsRelErr;
run;

proc means data=bin_fit n mean max;
    var AbsRelErr;
    output out=bre_stats
        mean = BRE_mean
        max  = BRE_max;
run;

proc print data=bre_stats;
run;
