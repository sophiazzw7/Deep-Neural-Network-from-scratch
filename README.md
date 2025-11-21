/*------------------------------------------------------------
  Step 1: Sort data and compute empirical and model CDF
-------------------------------------------------------------*/
proc sort data=severity_et7_10k out=sev_sorted_et7;
    by gross_loss;
run;

/* Get sample size */
proc sql noprint;
    select count(*) into :N_sev_et7 trimmed
    from sev_sorted_et7;
quit;

/*------------------------------------------------------------
  Step 2: Compute truncated mixture CDF and CvM (W^2) for ET7
-------------------------------------------------------------*/
data sev_cdf_et7;
    set sev_sorted_et7;
    retain i 0;
    i + 1;
    n = &N_sev_et7;

    /* Empirical CDF at this ordered point */
    F_emp = (i - 0.5) / n;

    /* Untruncated Exp CDF: 1 - exp(-x / sigma) */
    Fexp_x = 1 - exp( -gross_loss / &sigma_exp_et7 );
    Fexp_L = 1 - exp( -&L_trunc     / &sigma_exp_et7 );

    if gross_loss < &L_trunc then Fexp_tr = 0;
    else Fexp_tr = (Fexp_x - Fexp_L) / (1 - Fexp_L);

    /* Untruncated Lognormal CDF */
    Fln_x = cdf("LOGNORMAL", gross_loss, &intercept_ln_et7, &sdlog_ln_et7);
    Fln_L = cdf("LOGNORMAL", &L_trunc,   &intercept_ln_et7, &sdlog_ln_et7);

    if gross_loss < &L_trunc then Fln_tr = 0;
    else Fln_tr = (Fln_x - Fln_L) / (1 - Fln_L);

    /* Mixture CDF at this point */
    F_mix = &w1_et7 * Fexp_tr + &w2_et7 * Fln_tr;

    /* CvM contribution */
    diff  = F_mix - F_emp;
    diff2 = diff * diff;
run;

/* Sum of squared differences = Cramer–von Mises statistic W^2 */
proc means data=sev_cdf_et7 noprint;
    var diff2;
    output out=Obs_CvM_sev_et7 (drop=_:) sum = W2_obs_sev_et7;
run;

proc print data=Obs_CvM_sev_et7 label noobs;
    label W2_obs_sev_et7 = "Observed Cramer–von Mises Statistic (Severity ET7)";
run;
