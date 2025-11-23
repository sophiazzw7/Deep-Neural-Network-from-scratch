
data severity_et2_10k;
    set lee.cleaned_severity_data;
    if strip('Basel Event Type Level 1'n) = "ET2 - External Fraud"
       and gross_loss >= 10000;
run;

proc sql noprint;
    select count(*) into :n_obs trimmed
    from severity_et2_10k;
quit;

%let L_trunc           = 10000;

%let intercept_truncexp = 8.8993;
%let sigma_truncexp    = %sysevalf(%sysfunc(exp(&intercept_truncexp)));

%let intercept_lognorm = 11.0232;
%let scale_lognorm     = 0.6533;
%let sdlog_lognorm     = %sysevalf(%sysfunc(sqrt(&scale_lognorm)));

%let mix_p1            = 0.7606;   /* truncated exponential weight */
%let mix_p2            = 0.2394;   /* truncated lognormal weight  */

proc sort data=severity_et2_10k out=severity_et2_10k_sorted;
    by gross_loss;
run;

data AD_out;
    set severity_et2_10k_sorted end=last;
    /* Temporary array to store F(x_i) for all i */
    array Fv[&n_obs.] _temporary_;

    retain Ltrunc thetaE muL sigL w1 w2 F0E_L F0L_L;

    /* One-time setup on first observation */
    if _N_ = 1 then do;
        Ltrunc = &L_trunc.;
        thetaE = &sigma_truncexp.;
        muL    = &intercept_lognorm.;
        sigL   = &sdlog_lognorm.;
        w1     = &mix_p1.;
        w2     = &mix_p2.;

        /* Base CDFs at truncation point for each component */
        F0E_L = 1 - exp(-Ltrunc/thetaE);                  /* exponential CDF at L */
        F0L_L = cdf('LOGNORMAL', Ltrunc, muL, sigL);      /* lognormal CDF at L */
    end;

    i = _N_;   /* rank index */

    /* Base CDFs at this point (before truncation) */
    F0E_x = 1 - exp(-gross_loss/thetaE);
    F0L_x = cdf('LOGNORMAL', gross_loss, muL, sigL);

    /* Left-truncated CDFs (x >= Ltrunc) */
    F_E_tr = (F0E_x - F0E_L) / (1 - F0E_L);
    F_L_tr = (F0L_x - F0L_L) / (1 - F0L_L);

    /* Mixture CDF at x_i */
    F = w1*F_E_tr + w2*F_L_tr;

    /* Store F(x_i) in array */
    Fv[i] = F;

    /* On last obs, compute AD statistic A^2 */
    if last then do;
        n = &n_obs.;
        sumTerm = 0;

        do j = 1 to n;
            Fj      = Fv[j];
            Fj_comp = 1 - Fv[n + 1 - j];

            /* Guard against log(0) */
            if Fj      < 1e-12 then Fj      = 1e-12;
            if Fj      > 1-1e-12 then Fj    = 1-1e-12;
            if Fj_comp < 1e-12 then Fj_comp = 1e-12;
            if Fj_comp > 1-1e-12 then Fj_comp = 1-1e-12;

            sumTerm + (2*j - 1) * ( log(Fj) + log(Fj_comp) );
        end;

        A2 = -n - (1/n) * sumTerm;   /* Anderson–Darling statistic */

        output;
    end;

    keep n A2;
run;

proc print data=AD_out;
run;


Use `A2` as your AD metric for the severity mixture.
If you want, I can also plug KS and CvM into the same framework so you have **KS, CvM, AD** all computed from the *same* mixture CDF (all without IML).
