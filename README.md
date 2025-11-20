/*==========================================================
 STEP 2: Create bins for chi-square test
   Bins (operational-risk oriented):
   1:  0–120
   2: 121–160
   3: 161–220
   4: 221–350
   5: 351–600
   6: 601–900
   7: 901–max(frequency)
==========================================================*/

data freq_et2_bins;
    set freq_et2;
    length bin 8;

    if 0   <= frequency <= 120  then bin = 1;
    else if 121 <= frequency <= 160  then bin = 2;
    else if 161 <= frequency <= 220  then bin = 3;
    else if 221 <= frequency <= 350  then bin = 4;
    else if 351 <= frequency <= 600  then bin = 5;
    else if 601 <= frequency <= 900  then bin = 6;
    else if 901 <= frequency         then bin = 7;
run;

/* Observed counts per bin */
proc sql;
    create table obs_bins as
    select bin, count(*) as obs
    from freq_et2_bins
    group by bin
    order by bin;
quit;

/*==========================================================
 STEP 3: Expected counts from NB(&mu_hat, &r_hat, &p_hat)
==========================================================*/
proc iml;
    /* read all frequencies to get max value and N */
    use freq_et2_bins;
        read all var {frequency} into x;
    close freq_et2_bins;

    n      = nrow(x);
    r_hat  = &r_hat;        /* dispersion (size)   */
    p_hat  = &p_hat;        /* success probability */

    /* bin boundaries (last hi set to max(x))        */
    bin_lo = {0,   121, 161, 221, 351, 601, 901};
    bin_hi = {120, 160, 220, 350, 600, 900, max(x)};

    nbins = nrow(bin_lo);
    exp   = j(nbins,1,.);

    do i = 1 to nbins;
        lo = bin_lo[i];
        hi = bin_hi[i];

        prob = cdf("negbinomial", hi, r_hat, p_hat)
             - cdf("negbinomial", lo-1, r_hat, p_hat);

        exp[i] = n * prob;
    end;

    binIdx = t(1:nbins);
    create exp_bins var {"bin" "exp"};
        append from (binIdx || exp);
    close exp_bins;
quit;

/*==========================================================
 STEP 4: Chi-square statistic, df, p-value
==========================================================*/
proc sql;
    create table chi_input as
    select a.bin, a.obs, b.exp
    from obs_bins as a
    left join exp_bins as b
    on a.bin = b.bin
    order by a.bin;
quit;

data chi_sq_et2;
    set chi_input;
    chi_term = (obs - exp)**2 / exp;
run;

proc sql noprint;
    select count(*) into :k_bins
    from chi_sq_et2
    where exp > 0;

    select sum(chi_term) into :chi_stat
    from chi_sq_et2;
quit;

/* df = number of bins - number of estimated params (mu, r) - 1 */
%let df = %eval(&k_bins - 2 - 1);

data chi_result_et2;
    chi_square = &chi_stat;
    df         = &df;
    p_value    = 1 - probchi(chi_square, df);
run;

/* Optional: print details */
proc print data=chi_sq_et2 label;
    title "ET2 NB Chi-square – Observed vs Expected by Bin";
    label bin      = "Bin"
          obs      = "Observed Count"
          exp      = "Expected Count"
          chi_term = "Chi-square Contribution";
run;

proc print data=chi_result_et2 label noobs;
    title "ET2 NB Frequency – Overall Chi-square Goodness-of-Fit";
    label chi_square = "Chi-square Statistic"
          df         = "Degrees of Freedom"
          p_value    = "p-value";
run;
