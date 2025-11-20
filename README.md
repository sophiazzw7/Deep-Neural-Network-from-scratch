proc iml;
    /*-----------------------------
      1. Read raw ET2 frequencies
    ------------------------------*/
    use freq_et2;
        read all var {frequency} into x;
    close freq_et2;

    r_hat = &r_hat;
    p_hat = &p_hat;

    call randseed(12345);

    /*-----------------------------
      2. Helper: group frequencies
         into coarse bins and use
         bin midpoints
         (same spirit as chi-square)
         Bins:
         0–120   ->  60
         121–160 -> 140
         161–220 -> 190
         221–350 -> 285
         351–600 -> 475
         601–900 -> 750
         901+    -> 1300
    ------------------------------*/
    start GroupFreq(x);
        n = nrow(x);
        g = j(n,1,.);
        do i = 1 to n;
            xi = x[i];
            if      0   <= xi <= 120  then g[i] =  60;
            else if 121 <= xi <= 160  then g[i] = 140;
            else if 161 <= xi <= 220  then g[i] = 190;
            else if 221 <= xi <= 350  then g[i] = 285;
            else if 351 <= xi <= 600  then g[i] = 475;
            else if 601 <= xi <= 900  then g[i] = 750;
            else if 901 <= xi         then g[i] = 1300;
        end;
        return(g);
    finish;

    /*-----------------------------
      3. CvM with mid-CDF for NB
    ------------------------------*/
    start CvM_mid_stat(x, p_hat, r_hat);
        n = nrow(x);
        call sort(x,1);
        W2 = 0;
        do i = 1 to n;
            xi  = x[i];
            Flo  = cdf("NEGBINOMIAL", xi-1, p_hat, r_hat);
            Fhi  = cdf("NEGBINOMIAL", xi,   p_hat, r_hat);
            Fmid = 0.5*(Flo + Fhi);
            ui   = (2*i-1)/(2*n);
            W2   = W2 + (Fmid - ui)**2;
        end;
        W2 = W2 + 1/(12*n);
        return( W2 );
    finish;

    /*-----------------------------
      4. Observed grouped CvM
    ------------------------------*/
    xg = GroupFreq(x);                 /* group real data        */
    W0 = CvM_mid_stat(xg, p_hat, r_hat);

    /*-----------------------------
      5. Parametric bootstrap
         under NB(r_hat, p_hat),
         grouped the same way
    ------------------------------*/
    n   = nrow(x);
    B   = 1000;
    Wbt = j(B,1,.);

    do b = 1 to B;
        xSim = j(n,1,.);
        call randgen(xSim, "NEGBINOMIAL", p_hat, r_hat);
        xSimG = GroupFreq(xSim);
        Wbt[b] = CvM_mid_stat(xSimG, p_hat, r_hat);
    end;

    pval = mean(Wbt >= W0);

    print W0  label="Grouped CvM_mid (ET2)"
          pval label="Bootstrap p-value";

    create cvm_group_boot_et2 var {"W0" "pval"};
        append;
    close cvm_group_boot_et2;
quit;
