/* 1. Merge expected counts onto sim_counts_full by bin */
proc sort data=sim_counts_full; by bin; run;
proc sort data=bin_probs;       by bin; run;

data sim_counts_with_exp;
    merge sim_counts_full bin_probs(keep=bin expected);
    by bin;
run;

/* 2. Sort by sim_id bin for per-simulation chi-square */
proc sort data=sim_counts_with_exp; by sim_id bin; run;

/* 3. Compute chi-square per simulation */
data sim_chi;
    set sim_counts_with_exp;
    by sim_id;
    retain chisq 0;
    if first.sim_id then chisq = 0;
    chisq + (count_sim - expected)**2 / expected;
    if last.sim_id then output;
    keep sim_id chisq;
run;

proc print data=sim_chi(obs=10); run;  /* quick check */
