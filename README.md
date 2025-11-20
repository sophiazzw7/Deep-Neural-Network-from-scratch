proc iml;
    use freq_et2;
        read all var {frequency} into x;
    close freq_et2;

    n      = nrow(x);
    r_hat  = &r_hat;
    p_hat  = &p_hat;

    call sort(x, 1);         /* sort ascending */

    W2 = 0;
    do i = 1 to n;
        xi = x[i];
        F_lo  = cdf("NEGBINOMIAL", xi-1, p_hat, r_hat);
        F_hi  = cdf("NEGBINOMIAL", xi,   p_hat, r_hat);
        Fmid  = 0.5*(F_lo + F_hi);

        ui = (2*i-1)/(2*n);
        W2 = W2 + (Fmid - ui)**2;
    end;
    W2 = W2 + 1/(12*n);
    print W2[label="CvM_mid"];
quit;
