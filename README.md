data CvM_out;
    set severity_et2_10k_sorted end=last;
    retain i 0 sum_sq 0 max_diff 0
           F0E_L F0L_L Ltrunc thetaE muL sigL w1 w2;

    /* one-time setup on first obs */
    if i = 0 then do;
        Ltrunc = &L_trunc;
        thetaE = &sigma_truncexp;
        muL    = &intercept_lognorm;
        sigL   = &sdlog_lognorm;
        w1     = &mix_p1;
        w2     = &mix_p2;

        /* base CDFs at truncation point for each component */
        F0E_L = 1 - exp(-Ltrunc/thetaE);                     /* exponential CDF at L */
        F0L_L = cdf('LOGNORMAL', Ltrunc, muL, sigL);         /* lognormal CDF at L */
    end;

    i + 1;   /* rank index */

    /* CDF of each component at this gross_loss (before truncation) */
    F0E_x  = 1 - exp(-gross_loss/thetaE);
    F0L_x  = cdf('LOGNORMAL', gross_loss, muL, sigL);

    /* left-truncated CDFs (x >= Ltrunc) */
    F_E_tr = (F0E_x - F0E_L) / (1 - F0E_L);
    F_L_tr = (F0L_x - F0L_L) / (1 - F0L_L);

    /* mixture CDF */
    F = w1*F_E_tr + w2*F_L_tr;

    /* plotting position U_i = (2i-1)/(2n) */
    U = (2*i - 1) / (2 * &n_obs.);

    diff = F - U;
    sum_sq + diff*diff;

    if abs(diff) > max_diff then max_diff = abs(diff);

    /* on last obs, compute KS and CvM */
    if last then do;
        n  = &n_obs.;
        W2 = 1/(12*n) + sum_sq;   /* Cramér–von Mises */
        KS = max_diff;            /* KS from same F(x) */
        output;
    end;

    keep n W2 KS;
run;

proc print data=CvM_out;
run;
