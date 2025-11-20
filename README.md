/* ------------------------------------------------------------------ */
/*  CI for simulated quantiles using mean ± 1.96*SD across simulations */
/*    – one CI for each of Q_50, Q_75, Q_90, Q_95, Q_99               */
/* ------------------------------------------------------------------ */

/* 1. Get mean and SD of each simulated quantile across sim_id */
proc means data=sim_q noprint;
    var Q_50 Q_75 Q_90 Q_95 Q_99;
    output out=ci_stats
        mean = mean_Q50 mean_Q75 mean_Q90 mean_Q95 mean_Q99
        std  = sd_Q50   sd_Q75   sd_Q90   sd_Q95   sd_Q99;
run;

/* 2. Compute 95% normal-approximation CI: mean ± 1.96 * SD */
data ci_quantiles;
    set ci_stats;

    Q50_L = mean_Q50 - 1.96 * sd_Q50;
    Q50_U = mean_Q50 + 1.96 * sd_Q50;

    Q75_L = mean_Q75 - 1.96 * sd_Q75;
    Q75_U = mean_Q75 + 1.96 * sd_Q75;

    Q90_L = mean_Q90 - 1.96 * sd_Q90;
    Q90_U = mean_Q90 + 1.96 * sd_Q90;

    Q95_L = mean_Q95 - 1.96 * sd_Q95;
    Q95_U = mean_Q95 + 1.96 * sd_Q95;

    Q99_L = mean_Q99 - 1.96 * sd_Q99;
    Q99_U = mean_Q99 + 1.96 * sd_Q99;
run;

/* 3. (Optional) Print CI table */
proc print data=ci_quantiles noobs;
    var mean_Q50 Q50_L Q50_U
        mean_Q75 Q75_L Q75_U
        mean_Q90 Q90_L Q90_U
        mean_Q95 Q95_L Q95_U
        mean_Q99 Q99_L Q99_U;
run;
