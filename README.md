/*-----------------------------------------------------------
  STEP 1: Count observed frequencies for each x
-----------------------------------------------------------*/
proc sql;
    create table freq_counts as
    select frequency as x,
           count(*)   as n_x
    from freq_et2
    group by frequency
    order by x;
quit;

/*-----------------------------------------------------------
  STEP 2: Compute expected NB counts and root differences
-----------------------------------------------------------*/
data rootogram_data;
    set freq_counts;

    n_total = &N_et2;   /* total number of quarters for ET2 */

    /* NB pmf: pdf("NEGBINOMIAL", x, p_hat, r_hat) */
    exp_cnt   = n_total * pdf("NEGBINOMIAL", x, &p_hat, &r_hat);

    root_obs  = sqrt(n_x);
    root_exp  = sqrt(exp_cnt);

    /* hanging rootogram: difference from expected root-count */
    diff = root_obs - root_exp;
run;
/*-----------------------------------------------------------
  STEP 3: Hanging Rootogram – sqrt(Obs) - sqrt(Exp)
-----------------------------------------------------------*/
proc sgplot data=rootogram_data;
    title "ET2 Negative Binomial – Hanging Rootogram";
    vbarparm category=x response=diff / barwidth=0.8;
    refline 0 / axis=y lineattrs=(thickness=2);
    yaxis label="sqrt(Observed Count) - sqrt(Expected Count)";
    xaxis label="Frequency Value";
run;
