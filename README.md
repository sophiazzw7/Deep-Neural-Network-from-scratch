
libname lee "/sasdata/sasdata15/rqcral/Users/g17183";

/* Filter to ET2 - External Fraud and gross_loss >= 10000 */
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

/* Left truncation threshold */
%let L_trunc = 10000;

/* Component 1: Truncated Exponential */
%let intercept_truncexp = 8.8993;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp)));

/* Component 2: Lognormal */
%let intercept_lognorm = 11.0232;    /* meanlog */
%let scale_lognorm     = 0.6533;     /* variance of log */
%let sdlog_lognorm     = %sysevalf(%sysfunc(sqrt(&scale_lognorm)));

/* Mixture probabilities */
%let mix_p1 = 0.7606;    /* TruncExpo weight */
%let mix_p2 = 0.2394;    /* Lognormal weight */

/* Quantiles of the actual ET2 severity data */
proc univariate data=severity_et2_10k noprint;
    var gross_loss;
    output out=obs_q
        pctlpts = 50 75 90 95 99
        pctlpre = Q_;
run;

/* obs_q will have variables: Q_50 Q_75 Q_90 Q_95 Q_99 */
proc print data=obs_q; run;

/* Number of simulations */
%let nSim = 5000;

/* Sample size = number of observed ET2 losses */
proc sql noprint;
    select count(*) into :N_obs trimmed
    from severity_et2_10k;
quit;

/* Simulate nSim datasets of size N_obs from the truncated mixture */
data sim_sev;
    call streaminit(12345);
    do sim_id = 1 to &nSim;
        do i = 1 to &N_obs;
            u = rand("UNIFORM");
            /* Component 1: truncated exponential */
            if u < &mix_p1 then do;
                z = &L_trunc - 1;
                do while (z < &L_trunc);
                    z = rand("EXPONENTIAL", &sigma_truncexp);
                end;
            end;
            /* Component 2: truncated lognormal */
            else do;
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

/* Quantiles per simulation */
proc sort data=sim_sev; by sim_id; run;

proc univariate data=sim_sev noprint;
    by sim_id;
    var gross_loss_sim;
    output out=sim_q
        pctlpts = 50 75 90 95 99
        pctlpre = Q_;
run;

/* sim_q has one row per sim_id, with Q_50 Q_75 Q_90 Q_95 Q_99 */

/* 50th percentile (median) bands */
proc univariate data=sim_q noprint;
    var Q_50;
    output out=b_Q50
        pctlpts = 2.5 50 97.5
        pctlname = P2 P50 P97;
run;

/* 75th percentile bands */
proc univariate data=sim_q noprint;
    var Q_75;
    output out=b_Q75
        pctlpts = 2.5 50 97.5
        pctlname = P2 P50 P97;
run;

/* 90th percentile bands */
proc univariate data=sim_q noprint;
    var Q_90;
    output out=b_Q90
        pctlpts = 2.5 50 97.5
        pctlname = P2 P50 P97;
run;

/* 95th percentile bands */
proc univariate data=sim_q noprint;
    var Q_95;
    output out=b_Q95
        pctlpts = 2.5 50 97.5
        pctlname = P2 P50 P97;
run;

/* 99th percentile bands */
proc univariate data=sim_q noprint;
    var Q_99;
    output out=b_Q99
        pctlpts = 2.5 50 97.5
        pctlname = P2 P50 P97;
run;

/* Combine bands into one row */
data sim_bands;
    merge b_Q50(rename=(P2=P2_Q50 P50=P50_Q50 P97=P97_Q50))
          b_Q75(rename=(P2=P2_Q75 P50=P50_Q75 P97=P97_Q75))
          b_Q90(rename=(P2=P2_Q90 P50=P50_Q90 P97=P97_Q90))
          b_Q95(rename=(P2=P2_Q95 P50=P50_Q95 P97=P97_Q95))
          b_Q99(rename=(P2=P2_Q99 P50=P50_Q99 P97=P97_Q99));
run;

proc print data=sim_bands; run;
