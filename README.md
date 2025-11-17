/*-----------------------------------------------------------
  0. Load ET2 severity data (developer’s dataset)
-----------------------------------------------------------*/
libname lee "/sasdata/sasdata15/rqcral/Users/g17183";

data severity_et2_10k;
    set lee.severity;
    if Basel_Event_Type_Level_1 = "ET2 - External Fraud" and gross_loss >= 10000;
run;

/*-----------------------------------------------------------
  1. USE DEVELOPER'S FINAL PARAMETERS DIRECTLY
     NO PROC FMM IS NEEDED
-----------------------------------------------------------*/

/* Developer’s mixture parameters */
%let L_trunc = 10000;

/* Component 1: Truncated Exponential */
%let intercept_truncexp = 8.8993;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp)));

/* Component 2: Lognormal */
%let intercept_lognorm = 11.0232;    /* meanlog */
%let scale_lognorm     = 0.6533;     /* variance of lognormal */
%let sdlog_lognorm     = %sysevalf(%sysfunc(sqrt(&scale_lognorm)));

/* Mixing probabilities */
%let mix_p1 = 0.7606;    /* Truncated Exponential weight */
%let mix_p2 = 0.2394;    /* Lognormal weight */

/*-----------------------------------------------------------
  2. AD test for mixture distribution
-----------------------------------------------------------*/

proc datasets lib=work nolist;
    delete AD_Severity_Result AD_Severity_Bootstrap;
quit;

proc iml;

L      = &L_trunc;
p1     = &mix_p1;
p2     = &mix_p2;

sigmaE = &sigma_truncexp;     
thetaL = &intercept_lognorm;  
sigmaL = &sdlog_lognorm;

/* Mixture CDF with left truncation */
start F_mix(x);
    n = nrow(x);
    F = j(n,1,.);

    /* Truncation adjustment constants */
    F0E_L = cdf("EXPONENTIAL", L, sigmaE);
    F0L_L = cdf("LOGNORMAL",   L, thetaL, sigmaL);

    do i = 1 to n;
        xi = x[i];

        /* TruncExpo CDF */
        F0E   = cdf("EXPONENTIAL", xi, sigmaE);
        FexpT = (F0E - F0E_L) / (1 - F0E_L);

        /* Lognormal CDF */
        F0L   = cdf("LOGNORMAL", xi, thetaL, sigmaL);
        FlogT = (F0L - F0L_L) / (1 - F0L_L);

        Fi = p1*FexpT + p2*FlogT;

        if Fi < 1e-12 then Fi = 1e-12;
        if Fi > 1-1e-12 then Fi = 1-1e-12;

        F[i] = Fi;
    end;

    return(F);
finish;

/* AD statistic */
start AD_mix(x);
    call sort(x,1);
    n = nrow(x);
    F = F_mix(x);

    idx = T(1:n);
    A2 = -n - (1/n)*sum( (2*idx-1)#( log(F) + log(1 - F[n+1-idx]) ) );
    return(A2);
finish;

/* Load severity data */
use severity_et2_10k;
    read all var {gross_loss} into y;
close;

/* Observed AD statistic */
A2_obs = AD_mix(y);

/*-----------------------------------------------------------
  Bootstrap
-----------------------------------------------------------*/
n     = nrow(y);
nBoot = 10000;
call randseed(12345);

A2_boot = j(nBoot,1,.);

do b = 1 to nBoot;
    yb = j(n,1,.);

    do i = 1 to n;
        u = rand("UNIFORM");

        if u < p1 then do;
            /* Truncated exponential */
            z = L - 1;
            do while (z < L);
                z = rand("EXPONENTIAL", sigmaE);
            end;

        end;
        else do;
            z = L - 1;
            /* Truncated lognormal */
            do while (z < L);
                z = rand("LOGNORMAL", thetaL, sigmaL);
            end;
        end;

        yb[i] = z;
    end;

    A2_boot[b] = AD_mix(yb);
end;

/* p-value */
pval = (1 + sum(A2_boot >= A2_obs)) / (nBoot + 1);

/* Save result */
results = A2_obs || pval || p1 || p2 || nBoot;
create AD_Severity_Result from results
    [colname={"A2_obs" "pval" "pi_exp" "pi_logn" "B"}];
append from results;
close AD_Severity_Result;

create AD_Severity_Bootstrap from A2_boot[colname={"A2_boot"}];
append from A2_boot;
close AD_Severity_Bootstrap;

quit;

/* Print summary */
proc print data=AD_Severity_Result noobs label;
    label A2_obs = "Observed AD Statistic"
          pval   = "Bootstrap p-value"
          pi_exp = "TruncExpo weight"
          pi_logn= "Lognormal weight"
          B      = "Bootstrap reps";
run;

proc means data=AD_Severity_Bootstrap min p25 p50 p75 max;
    var A2_boot;
run;
