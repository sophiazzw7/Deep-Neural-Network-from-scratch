/*==========================================================
   DENSITY RATIO PLOT FOR ET2 FREQUENCY — SAS CODE
==========================================================*/

/* Step 1: Compute observed probabilities for each distinct x */
proc sql;
    create table freq_counts as
    select frequency as x,
           count(*) as n_x
    from freq_et2
    group by frequency
    order by frequency;
quit;

data freq_counts;
    set freq_counts;
    n_total = &N_et2;
    p_obs = n_x / n_total;
run;

/* Step 2: Compute NB expected pmf for each x */
data freq_ratio;
    set freq_counts;

    /* NB PMF syntax: pdf("NEGBINOMIAL", x, p_hat, r_hat) */
    p_exp = pdf("NEGBINOMIAL", x, &p_hat, &r_hat);

    /* Density Ratio R(x) */
    ratio = p_obs / p_exp;
run;

/* Step 3: Plot Ratio vs Frequency */
proc sgplot data=freq_ratio;
    title "ET2 Negative Binomial – Density Ratio Plot (Observed / Expected)";
    scatter x=x y=ratio / markerattrs=(symbol=circlefilled size=7);
    series x=x y=ratio / lineattrs=(pattern=solid);
    refline 1 / axis=y lineattrs=(color=red thickness=2); /* perfect-fit line */
    yaxis label="Observed / Expected (Density Ratio)" min=0;
    xaxis label="Frequency Value";
run;
