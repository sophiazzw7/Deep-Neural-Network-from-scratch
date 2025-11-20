data freq_w_cdf;
    set freq_et2;
    /* Compute Theoretical CDF: F(x_i) using estimated parameters */
    /* CDF('NEGBINOMIAL', x, p, r) returns Prob(X <= x) */
    F_theo = cdf('NEGBINOMIAL', frequency, &p_hat, &r_hat);
run;

/* 3. SORT DATA (Crucial Step for CvM) */
/* The formula requires data to be ordered: x_(1) <= x_(2) ... */
proc sort data=freq_w_cdf;
    by frequency;
run;

/* --------------------------------------------------------- */
/* 4. COMPUTE CVM STATISTIC (W^2)                            */
/* --------------------------------------------------------- */
data cvm_result;
    set freq_w_cdf end=last;
    
    /* Rank i is simply the current row number (_N_) */
    rank_i = _N_;
    N = &N_obs;
    
    /* Calculate the "Empirical Height" at midpoint: (i - 0.5) / N */
    emp_cdf_mid = (rank_i - 0.5) / N;
    
    /* Squared difference term */
    sq_diff = (F_theo - emp_cdf_mid)**2;
    
    /* Accumulate the Sum */
    retain sum_sq 0;
    sum_sq = sum_sq + sq_diff;
    
    /* On the last row, apply the final correction: 1/(12N) */
    if last then do;
        CvM_Stat = sum_sq + (1 / (12 * N));
        put "Calculated Cramér–von Mises Statistic (W^2): " CvM_Stat;
        output;
    end;
    
    keep CvM_Stat;
run;

/* View Result */
proc print data=cvm_result; run;
