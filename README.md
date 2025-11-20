/*------------------------------------------------------------
  0. Prep ET7 severity data (>= 10k)
------------------------------------------------------------*/
libname ops "/sasdata/mrmg2/users/G07267/MOD13638_2025/Output";

data severity_et7_10k;
    set ops.cleaned_severity_data;
    if strip('Basel Event Type Level 1'n) = "ET7 - Execution Delivery and Process Management"
       and gross_loss >= 10000;
run;

%let L_trunc = 10000;

/*------------------------------------------------------------
  1. Mixture parameters from model doc
------------------------------------------------------------*/
%let intercept_truncexp = 9.4179;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp.)));

%let intercept_lognorm  = 11.6984;
%let scale_lognorm      = 1.6015;
%let sdlog_lognorm      = %sysevalf(%sysfunc(sqrt(&scale_lognorm.)));

%let mix_p1 = 0.7036;
%let mix_p2 = 0.2964;

/* Sample size */
proc sql noprint;
    select count(*) into :N_obs trimmed
    from severity_et7_10k;
quit;
%put NOTE: N_obs=&N_obs.;

/*------------------------------------------------------------
  2. Get empirical quantiles to define bins
     Bins: [L_trunc,Q50), [Q50,Q75), [Q75,Q90),
           [Q90,Q95), [Q95, +inf)
------------------------------------------------------------*/
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

/*------------------------------------------------------------
  3. Function: mixture CDF at a value x (>= L_trunc)
------------------------------------------------------------*/
%macro mix_cdf(x);
    /* Truncated exponential above L_trunc */
    %if &x. < &L_trunc. %then 0;
    %else %do;
        /* Exponential part */
        %let Fexp = cdf('EXPONENTIAL', %sysevalf(&x.-&L_trunc.), &sigma_truncexp.);

        /* Lognormal part truncated at L_trunc */
        %let FL  = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        %let F0  = cdf('LOGNORMAL', &x.,        &intercept_lognorm., &sdlog_lognorm.);

        %if &FL. >= 0.999999 %then %let Flog = 0;
        %else %let Flog = %sysevalf((&F0.-&FL.)/(1-&FL.));

        %sysevalf(&mix_p1.*&Fexp. + &mix_p2.*&Flog.)
    %end;
%mend;

/*------------------------------------------------------------
  4. Compute model probabilities for the 5 bins
------------------------------------------------------------*/
/* Bin boundaries (lower inclusive, upper exclusive except last) */
data bin_probs;
    length bin 8;
    F_L = .; F_U = .; P_model = .;

    /* Bin 1: [L_trunc, Q50) */
    bin = 1;
    F_L = %mix_cdf(&L_trunc.);
    F_U = %mix_cdf(&Q50.);
    P_model = F_U - F_L;
    output;

    /* Bin 2: [Q50, Q75) */
    bin = 2;
    F_L = %mix_cdf(&Q50.);
    F_U = %mix_cdf(&Q75.);
    P_model = F_U - F_L;
    output;

    /* Bin 3: [Q75, Q90) */
    bin = 3;
    F_L = %mix_cdf(&Q75.);
    F_U = %mix_cdf(&Q90.);
    P_model = F_U - F_L;
    output;

    /* Bin 4: [Q90, Q95) */
    bin = 4;
    F_L = %mix_cdf(&Q90.);
    F_U = %mix_cdf(&Q95.);
    P_model = F_U - F_L;
    output;

    /* Bin 5: [Q95, +inf) */
    bin = 5;
    F_L = %mix_cdf(&Q95.);
    F_U = 1;
    P_model = F_U - F_L;
    output;
run;

/* Expected counts per bin */
data bin_probs;
    set bin_probs;
    E_count = &N_obs. * P_model;
run;

/*------------------------------------------------------------
  5. Observed counts per bin
------------------------------------------------------------*/
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

/*------------------------------------------------------------
  6. Merge and compute Binned Relative Error metrics
------------------------------------------------------------*/
data bin_fit;
    merge bin_probs bin_obs;
    by bin;
    if E_count > 0 then RelErr = (O_count - E_count) / E_count;
    AbsRelErr = abs(RelErr);
run;

proc means data=bin_fit n mean max;
    var AbsRelErr;
    output out=bre_stats
        mean = BRE_mean
        max  = BRE_max;
run;

proc print data=bin_fit;
    var bin O_count E_count RelErr AbsRelErr;
run;

proc print data=bre_stats;
run;
