/*---------------------------------------------------------
* 1. Merge actual quantiles and simulation bands
*--------------------------------------------------------*/
data quant_ci_check;
    merge obs_q  /* actual: Q_50 Q_75 Q_90 Q_95 Q_99 */
          sim_bands;  /* bands: Q50_L Q50_U ... Q99_L Q99_U */
    /* no BY since each has 1 row */
run;

/*---------------------------------------------------------
* 2. Apply one-sided upper test with 5% tolerance
*    - Fail only if Actual > (1+tol)*Upper
*    - Q99 is diagnostic only (no pass flag)
*--------------------------------------------------------*/
%let tol = 0.05;   /* 5% tolerance */

data quant_ci_result;
    set quant_ci_check;

    /* Midpoints & relative deviations (optional – for reporting) */
    Q50_mid = (Q50_L + Q50_U)/2;
    Q75_mid = (Q75_L + Q75_U)/2;
    Q90_mid = (Q90_L + Q90_U)/2;
    Q95_mid = (Q95_L + Q95_U)/2;
    Q99_mid = (Q99_L + Q99_U)/2;

    Q50_rel_dev = (Q_50 - Q50_mid) / Q50_mid;
    Q75_rel_dev = (Q_75 - Q75_mid) / Q75_mid;
    Q90_rel_dev = (Q_90 - Q90_mid) / Q90_mid;
    Q95_rel_dev = (Q_95 - Q95_mid) / Q95_mid;
    Q99_rel_dev = (Q_99 - Q99_mid) / Q99_mid;

    /* One-sided upper tests with tolerance for 50–95 */
    Q50_pass = (Q_50 <= (1+&tol.)*Q50_U);
    Q75_pass = (Q_75 <= (1+&tol.)*Q75_U);
    Q90_pass = (Q_90 <= (1+&tol.)*Q90_U);
    Q95_pass = (Q_95 <= (1+&tol.)*Q95_U);

    /* Q99 is diagnostic only – no pass/fail flag.
       If you still want to see whether it exceeds the band+tol: */
    Q99_exceeds = (Q_99 > (1+&tol.)*Q99_U);

    /* Overall flag for the formal test (Q50–Q95 only) */
    overall_pass = (Q50_pass and Q75_pass and Q90_pass and Q95_pass);
run;

proc print data=quant_ci_result;
    var Q_50 Q50_L Q50_U Q50_pass
        Q_75 Q75_L Q75_U Q75_pass
        Q_90 Q90_L Q90_U Q90_pass
        Q_95 Q95_L Q95_U Q95_pass
        Q_99 Q99_L Q99_U Q99_exceeds
        overall_pass;
run;
