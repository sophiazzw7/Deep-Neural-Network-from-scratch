proc iml;
    /* Read data */
    use freq_et2_bins;
        read all var {frequency} into x;
    close freq_et2_bins;

    n      = nrow(x);
    r_hat  = &r_hat;          /* size from your earlier code        */
    p_hat  = &p_hat;          /* success prob from your earlier code*/

    /* Bin boundaries */
    bin_lo = {0,   121, 161, 221, 351, 601, 901};
    bin_hi = {120, 160, 220, 350, 600, 900, 0};
    bin_hi[7] = max(x);       /* last bin goes to max observed freq */

    nbins = nrow(bin_lo);
    exp   = j(nbins, 1, .);

    do i = 1 to nbins;
        lo = bin_lo[i];
        hi = bin_hi[i];

        /* CDF syntax: CDF('NEGBINOMIAL', m, p, n)
           m = count, p = p_hat, n = r_hat (size) */

        if lo = 0 then
            prob = cdf("NEGBINOMIAL", hi, p_hat, r_hat);
        else
            prob = cdf("NEGBINOMIAL", hi, p_hat, r_hat)
                 - cdf("NEGBINOMIAL", lo-1, p_hat, r_hat);

        exp[i] = n * prob;
    end;

    /* Build output (bin, exp) for merge with obs */
    out = (T(1:nbins)) || exp;
    create exp_bins from out[colname={"bin" "exp"}];
        append from out;
    close exp_bins;
quit;
