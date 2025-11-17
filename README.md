
libname lee "/sasdata/sasdata15/rqcral/Users/g17183";

/* ET2 - External Fraud, severity >= 10,000 */
data severity_et2_10k;
    set lee.severity;
    if upcase(strip('Basel Event Type Level 1'n)) = "ET2 - EXTERNAL FRAUD"
       and gross_loss >= 10000;
run;

/* Sanity check */
proc sql;
    select count(*) as n_obs,
           min(gross_loss) as min_loss,
           max(gross_loss) as max_loss
    from severity_et2_10k;
quit;

/* Left truncation */
%let L_trunc = 10000;

/* Component 1: Truncated Exponential */
%let intercept_truncexp = 8.8993;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp)));

/* Component 2: Lognormal */
%let intercept_lognorm = 11.0232;    /* meanlog */
%let scale_lognorm     = 0.6533;     /* variance of log */
%let sdlog_lognorm     = %sysevalf(%sysfunc(sqrt(&scale_lognorm)));

/* Mixture weights */
%let mix_p1 = 0.7606;   /* TruncExpo weight */
%let mix_p2 = 0.2394;   /* Lognormal weight */

/* Number of simulations */
%let nSim   = 5000;

/* Define 6 bins: [10k,25k), [25k,50k), [50k,100k), [100k,250k), [250k,500k), [500k,+inf) */
data bin_def;
    length bin 8;
    input bin lower upper;
    datalines;
1 10000 25000
2 25000 50000
3 50000 100000
4 100000 250000
5 250000 500000
6 500000 .       /* . means open-ended top bin */
;
run;

/* Attach a bin to each observed loss */
data severity_binned;
    set severity_et2_10k;
    length bin 8;
    if      gross_loss < 25000  then bin = 1;
    else if gross_loss < 50000  then bin = 2;
    else if gross_loss < 100000 then bin = 3;
    else if gross_loss < 250000 then bin = 4;
    else if gross_loss < 500000 then bin = 5;
    else                             bin = 6;
run;

/* Observed counts per bin */
proc summary data=severity_binned nway;
    class bin;
    output out=obs_bins (drop=_type_ _freq_) n=count_obs;
run;

/* Total sample size */
proc sql noprint;
    select sum(count_obs) into :N_obs trimmed
    from obs_bins;
quit;
%put NOTE: N_obs = &N_obs.;

/* Compute model bin probabilities p_i and expected counts E_i = N_obs * p_i */
data bin_probs;
    if _n_=1 then do;
        /* Truncation adjustments */
        F0E_L = cdf("EXPONENTIAL", &L_trunc, &sigma_truncexp);
        F0L_L = cdf("LOGNORMAL",   &L_trunc, &intercept_lognorm, &sdlog_lognorm);
    end;

    set bin_def;
    /* Lower CDF */
    if lower < &L_trunc then lower=&L_trunc;

    /* F_mix at lower bound */
    F0E_low   = cdf("EXPONENTIAL", lower, &sigma_truncexp);
    FexpT_low = (F0E_low - F0E_L) / (1 - F0E_L);

    F0L_low   = cdf("LOGNORMAL",   lower, &intercept_lognorm, &sdlog_lognorm);
    FlogT_low = (F0L_low - F0L_L) / (1 - F0L_L);

    F_low = &mix_p1*FexpT_low + &mix_p2*FlogT_low;

    /* F_mix at upper bound (or 1 if open-ended) */
    if upper = . then do;
        F_high = 1;
    end;
    else do;
        F0E_high   = cdf("EXPONENTIAL", upper, &sigma_truncexp);
        FexpT_high = (F0E_high - F0E_L) / (1 - F0E_L);

        F0L_high   = cdf("LOGNORMAL",   upper, &intercept_lognorm, &sdlog_lognorm);
        FlogT_high = (F0L_high - F0L_L) / (1 - F0L_L);

        F_high = &mix_p1*FexpT_high + &mix_p2*FlogT_high;
    end;

    p_bin   = F_high - F_low;
    expected = &N_obs * p_bin;

    keep bin lower upper p_bin expected;
run;

proc print data=bin_probs; run;

/* Merge observed counts with expected counts, fill missing with 0 */
proc sort data=obs_bins;     by bin; run;
proc sort data=bin_probs;    by bin; run;

data obs_chi;
    merge bin_probs obs_bins;
    by bin;
    if count_obs = . then count_obs = 0;

    chisq_component = (count_obs - expected)**2 / expected;
run;

/* Total chi-square statistic for observed data */
proc summary data=obs_chi nway;
    var chisq_component;
    output out=obs_chi_total (drop=_type_ _freq_) sum=chisq_obs;
run;

proc print data=obs_chi_total; run;

/* Simulate nSim datasets of size N_obs from the fitted mixture */
data sim_sev;
    call streaminit(12345);
    do sim_id = 1 to &nSim;
        do i = 1 to &N_obs;
            u = rand("UNIFORM");
            if u < &mix_p1 then do;        /* Truncated exponential */
                z = &L_trunc - 1;
                do while (z < &L_trunc);
                    z = rand("EXPONENTIAL", &sigma_truncexp);
                end;
            end;
            else do;                       /* Truncated lognormal */
                z = &L_trunc - 1;
                do while (z < &L_trunc);
                    z = rand("LOGNORMAL", &intercept_lognorm, &sdlog_lognorm);
                end;
            end;
            gross_loss_sim = z;
            output;
        end;
    end;
    drop i u z;
run;

/* Bin simulated data the same way */
data sim_binned;
    set sim_sev;
    length bin 8;
    if      gross_loss_sim < 25000  then bin = 1;
    else if gross_loss_sim < 50000  then bin = 2;
    else if gross_loss_sim < 100000 then bin = 3;
    else if gross_loss_sim < 250000 then bin = 4;
    else if gross_loss_sim < 500000 then bin = 5;
    else                                 bin = 6;
run;

/* Counts by sim_id and bin */
proc summary data=sim_binned nway;
    class sim_id bin;
    output out=sim_counts (drop=_type_ _freq_) n=count_sim;
run;

/* Ensure every sim_id has all 6 bins (fill missing counts with 0) */
data template;
    do sim_id = 1 to &nSim;
        do bin = 1 to 6;
            output;
        end;
    end;
run;

proc sort data=template;    by sim_id bin; run;
proc sort data=sim_counts;  by sim_id bin; run;

data sim_counts_full;
    merge template sim_counts;
    by sim_id bin;
    if count_sim = . then count_sim = 0;
run;

/* Add expected counts and compute chi-square per simulation */
proc sort data=bin_probs; by bin; run;

data sim_chi;
    merge sim_counts_full bin_probs(keep=bin expected);
    by bin;
    retain chisq 0;
    if first.sim_id then chisq = 0;
    chisq + (count_sim - expected)**2 / expected;
    if last.sim_id then output;
    keep sim_id chisq;
run;

proc print data=sim_chi(obs=10); run;  /* preview first 10 */

/* Get observed chi-square value into macro variable */
proc sql noprint;
    select chisq_obs into :chisq_obs trimmed from obs_chi_total;
quit;
%put NOTE: Observed chi-square = &chisq_obs.;

/* Compute p-value: proportion of simulations with chi-square >= observed */
data pval;
    set sim_chi end=last;
    retain ge_count 0 total 0;
    total + 1;
    if chisq >= &chisq_obs then ge_count + 1;
    if last then do;
        p_value = ge_count / total;
        output;
    end;
    keep ge_count total p_value;
run;

proc print data=pval; run;
