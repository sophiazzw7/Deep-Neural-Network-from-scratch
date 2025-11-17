/*-----------------------------------------------------------
  0. Set up data for ET2 severity (gross_loss >= 10000)
-----------------------------------------------------------*/
libname lee "/sasdata/sasdata15/rqcral/Users/g17183";

data severity_et2_10k;
    set lee.severity;
    if Basel_Event_Type_Level_1 = "ET2 - External Fraud" and gross_loss >= 10000;
    format date_month_year monyy7.;
    date_month_year = mdy(month(data_date), 1, year(data_date));
run;

/*-----------------------------------------------------------
  1. Fit mixture model (developer’s model) – optional
     This is here just to reproduce their fit; AD test uses
     the macro values defined below.
-----------------------------------------------------------*/
ods graphics on;
proc fmm data=severity_et2_10k tech=newrap maxit=5000 gconv=0;
    model gross_loss = / dist=TruncExpo(10000,.);
    model            / dist=LogNormal;
run; quit;
ods graphics off;

/*-----------------------------------------------------------
  2. Parameters from the FMM output (copy from developer)
     Update these if the FMM fit is rerun and changes.
-----------------------------------------------------------*/

/* Truncated exponential component */
%let intercept_from_FMM_exponential = 8.8993;
%let sigma_for_rand_exponential =
    %sysevalf(%sysfunc(exp(&intercept_from_FMM_exponential)));  /* scale parameter */

/* Lognormal component */
%let intercept_from_FMM_lognormal = 11.0232;
%let scale_from_FMM_lognormal     = 0.6533;   /* variance on log scale */

%let theta_for_rand_lognormal  = &intercept_from_FMM_lognormal;   /* meanlog */
%let lambda_for_rand_lognormal =
    %sysevalf(%sysfunc(sqrt(&scale_from_FMM_lognormal)));         /* sdlog */

/* Mixture probabilities */
%let mixing_probability_model1 = 0.7606;   /* TruncExpo weight */
%let mixing_probability_model2 = 0.2394;   /* Lognormal weight */

/*-----------------------------------------------------------
  3. Anderson–Darling test for the truncated mixture
-----------------------------------------------------------*/

%let L_trunc = 10000;  /* left truncation threshold */

/* Cleanup old work tables, if any */
proc datasets lib=work nolist;
    delete AD_Severity_Result AD_Severity_Bootstrap;
quit;

proc iml;

L      = &L_trunc;
p1     = &mixing_probability_model1;
p2     = &mixing_probability_model2;
sigmaE = &sigma_for_rand_exponential;
thetaL = &theta_for_rand_lognormal;
sigmaL = &lambda_for_rand_lognormal;

/* Mixture CDF with left truncation at L */
start F_mix(x);
    n = nrow(x);
    F = j(n,1,.);

    /* Truncation adjustments */
    F0E_L = cdf("EXPONENTIAL", L, sigmaE);
    F0L_L = cdf("LOGNORMAL",   L, thetaL, sigmaL);

    do i = 1 to n;
        xi = x[i];

        F0E   = cdf("EXPONENTIAL", xi, sigmaE);
        FexpT = (F0E - F0E_L) / (1 - F0E_L);

        F0L   = cdf("LOGNORMAL", xi, thetaL, sigmaL);
        FlogT = (F0L - F0L_L) / (1 - F0L_L);

        Fi = p1*FexpT + p2*FlogT;

        if Fi < 1e-12 then Fi = 1e-12;
        if Fi > 1-1e-12 then Fi = 1-1e-12;

        F[i] = Fi;
    end;

    return(F);
finish;

/* Anderson–Darling statistic for the mixture */
start AD_mix(x);
    call sort(x, 1);
    n  = nrow(x);
    F  = F_mix(x);

    idx = T(1:n);
    A2 = -n - (1/n) * sum( (2*idx-1)#( log(F) + log(1 - F[n+1-idx]) ) );
    return(A2);
finish;

/* Read observed severity data */
use severity_et2_10k;
    read all var {gross_loss} into y;
close;

/* Observed AD statistic */
A2_obs = AD_mix(y);

/*-----------------------------------------------------------
  4. Parametric bootstrap from the same truncated mixture
-----------------------------------------------------------*/
n     = nrow(y);
nBoot = 10000;
call randseed(12345);

A2_boot = j(nBoot,1,.);

do b = 1 to nBoot;
    yb = j(n,1,.);

    do i = 1 to n;
        u = rand("UNIFORM");

        if u < p1 then do;   /* truncated exponential */
            z = L - 1;
            do while (z < L);
                z = rand("EXPONENTIAL", sigmaE);
            end;
        end;
        else do;             /* truncated lognormal */
            z = L - 1;
            do while (z < L);
                z = rand("LOGNORMAL", thetaL, sigmaL);
            end;
        end;

        yb[i] = z;
    end;

    A2_boot[b] = AD_mix(yb);
end;

/* Upper-tail bootstrap p-value */
pval = (1 + sum(A2_boot >= A2_obs)) / (nBoot + 1);

/*-----------------------------------------------------------
  5. Output AD results
-----------------------------------------------------------*/
results = A2_obs || pval || p1 || p2 || nBoot;

create AD_Severity_Result from results
    [colname={"A2_obs" "pval" "pi_exp" "pi_logn" "B"}];
append from results;
close AD_Severity_Result;

create AD_Severity_Bootstrap from A2_boot[colname={"A2_boot"}];
append from A2_boot;
close AD_Severity_Bootstrap;

quit;

/*-----------------------------------------------------------
  6. Print summary and bootstrap distribution
-----------------------------------------------------------*/
proc print data=AD_Severity_Result noobs label;
    label A2_obs = "AD Statistic (Observed)"
          pval   = "Bootstrap p-value"
          pi_exp = "Mixture weight: TruncExpo"
          pi_logn= "Mixture weight: Lognormal"
          B      = "Bootstrap reps";
run;

proc means data=AD_Severity_Bootstrap min p25 p50 p75 max;
    var A2_boot;
run;
