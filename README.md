proc iml;
    use freq_et2;
        read all var {frequency} into x;
    close freq_et2;

    start CvM_mid_stat(x, p_hat, r_hat);
        n = nrow(x);
        call sort(x,1);
        W2 = 0;
        do i = 1 to n;
            xi   = x[i];
            Flo  = cdf("NEGBINOMIAL", xi-1, p_hat, r_hat);
            Fhi  = cdf("NEGBINOMIAL", xi,   p_hat, r_hat);
            Fmid = 0.5*(Flo+Fhi);
            ui   = (2*i-1)/(2*n);
            W2  += (Fmid-ui)**2;
        end;
        return( W2 + 1/(12*n) );
    finish;

    n      = nrow(x);
    r_hat  = &r_hat;
    p_hat  = &p_hat;

    /* observed CvM */
    W0 = CvM_mid_stat(x, p_hat, r_hat);

    /* bootstrap */
    B = 1000;
    Wboot = j(B,1,.);
    do b = 1 to B;
        xSim = j(n,1,.);
        call randgen(xSim, "NEGBINOMIAL", p_hat, r_hat);
        Wboot[b] = CvM_mid_stat(xSim, p_hat, r_hat);
    end;

    pval = mean(Wboot >= W0);
    print W0[label="CvM_mid_obs"] pval[label="p_value"];
quit;
