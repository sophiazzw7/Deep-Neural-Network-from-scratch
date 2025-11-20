/* Compute mean and sd for simulated Q50 */
proc means data=sim_q noprint;
    var Q_50;
    output out=CI_Q50_mean
        mean = mean_Q50
        std  = sd_Q50;
run;

/* Create the Normal-approximation 95% CI */
data Q50_Normal_CI;
    set CI_Q50_mean;
    Lower_CI = mean_Q50 - 1.96 * sd_Q50;
    Upper_CI = mean_Q50 + 1.96 * sd_Q50;
run;

/* Print the CI */
proc print data=Q50_Normal_CI noobs;
    var mean_Q50 sd_Q50 Lower_CI Upper_CI;
    format mean_Q50 sd_Q50 Lower_CI_
