/* Q50 band across simulations */
proc univariate data=sim_each noprint;
    var P_50;
    output out=band50
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q50 P50_Q50 P97_Q50;
run;

/* Q75 band */
proc univariate data=sim_each noprint;
    var P_75;
    output out=band75
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q75 P50_Q75 P97_Q75;
run;

/* Q90 band */
proc univariate data=sim_each noprint;
    var P_90;
    output out=band90
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q90 P50_Q90 P97_Q90;
run;

/* Q95 band */
proc univariate data=sim_each noprint;
    var P_95;
    output out=band95
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q95 P50_Q95 P97_Q95;
run;

/* Q99 band */
proc univariate data=sim_each noprint;
    var P_99;
    output out=band99
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q99 P50_Q99 P97_Q99;
run;

/* Combine into one row */
data sim_band;
    merge band50 band75 band90 band95 band99;
run;
data freq_quantile_test;
    merge emp_freq sim_band;   /* both are single-row datasets */
run;

proc print data=freq_quantile_test;
run;
