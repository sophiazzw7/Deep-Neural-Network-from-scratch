/*-------------------------------------------------------------*/
/* 0. Subset to ET2 frequency data                             */
/*    Assumes ops.ef_dataset has:                              */
/*      - frequency                                            */
/*      - 'Basel Event Type Level 1'n                          */
/*-------------------------------------------------------------*/

data freq_et2;
    set ops.ef_dataset;
    /* Adjust this filter if your ET2 label is slightly different */
    if strip('Basel Event Type Level 1'n) = "ET2 - External Fraud";
    keep frequency;
run;

/*-------------------------------------------------------------*/
/* 1. Fit Negative Binomial with PROC COUNTREG                 */
/*    Model: frequency ~ intercept only, dist=NEGBIN           */
/*-------------------------------------------------------------*/

proc countreg data=freq_et2;
    model frequency = / dist=NEGBIN;
    ods output ParameterEstimates = nb_parms;
run;

/* Extract intercept and alpha from COUNTREG */
proc sql noprint;
    select estimate into :b0 trimmed
    from nb_parms
    where upcase(parameter) = 'INTERCEPT';

    select estimate into :alpha trimmed
    from nb_parms
    where upcase(parameter) like '%ALPHA%';
quit;

/* Convert to NB parameters used by CDF/RAND:
     mu = exp(b0)
     r  = 1/alpha
     p  = r / (r + mu)
*/
%let mu_hat = %sysevalf(exp(&b0));
%let r_hat  = %sysevalf(1/&alpha);
%let p_hat  = %sysevalf(&r_hat / (&r_hat + &mu_hat));

/*-------------------------------------------------------------*/
/* 2. Compute CvM statistics in PROC IML                       */
/*    A: CDF vs empirical CDF                                  */
/*    B: PIT-Uniform(0,1) classic CvM                          */
/*    C: Grouped-by-count discrete CvM                         */
/*-------------------------------------------------------------*/

proc iml;
    /* Read observed frequencies into a vector y */
    use freq_et2;
        read all var {frequency} into y;
    close freq_et2;

    n = nrow(y);
    p_hat = &p_hat;
    r_hat = &r_hat;

    /**************** A. CvM on CDF vs empirical CDF ****************/
    /* Sort y, get F_emp(i) = (i - 0.5)/n, F_mod from NB CDF */
    yA = y;                    /* work copy */
    call sort(yA, 1);          /* ascending */

    idx = T(1:n);              /* 1,2,...,n */
    F_emp = (idx - 0.5) / n;
    F_mod = cdf("NEGBINOMIAL", yA, p_hat, r_hat);

    diffA = F_mod - F_emp;
    W2_A  = diffA` * diffA;    /* sum of squared differences */

    /**************** B. CvM in PIT space (Uniform(0,1)) ************/
    /* U_i = F_mod(y_i); sort U; use textbook CvM formula */
    U = cdf("NEGBINOMIAL", y, p_hat, r_hat);
    call sort(U, 1);           /* sort PITs */

    target = ((2*idx) - 1) / (2*n);  /* (2i-1)/(2n) */
    diffB  = (U - target)##2;
    W2_B   = sum(diffB) + 1/(12*n);

    /**************** C. Grouped discrete-support CvM **************/
    /* Collapse duplicates of y, weight by frequency/n */
    yC = y;
    call sort(yC, 1);
    u = unique(yC);            /* unique support values */
    m = nrow(u);

    freq = j(m,1,0);           /* counts per unique value */

    do j = 1 to m;
        val      = u[j];
        idx_val  = loc(yC = val);
        freq[j]  = ncol(idx_val);   /* how many times val appears */
    end;

    cum      = cusum(freq);          /* cumulative counts */
    F_emp_g  = cum / n;              /* empirical CDF at each unique val */
    F_mod_g  = cdf("NEGBINOMIAL", u, p_hat, r_hat);
    weights  = freq / n;             /* probability mass at each val */

    W2_C = sum( weights # ((F_mod_g - F_emp_g)##2) );

    /**************** Print results ********************************/
    print n[label="Sample Size (n)"],
          p_hat r_hat[label="r (# of successes)"]
          ;

    print W2_A[label="CvM (A) - CDF vs empirical CDF"],
          W2_B[label="CvM (B) - PIT Uniform(0,1)"],
          W2_C[label="CvM (C) - Grouped discrete support"];
quit;
