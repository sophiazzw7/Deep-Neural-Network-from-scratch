/* 1. SETUP: Define Parameters exactly as in your screenshots */
%let L_trunc = 10000;

/* Component 1: Exponential Parameters */
%let intercept_truncexp = 9.4179;
%let sigma_truncexp = %sysevalf(%sysfunc(exp(&intercept_truncexp)));

/* Component 2: Lognormal Parameters */
%let intercept_lognorm = 11.6984; 
%let scale_lognorm = 1.6015;
%let sdlog_lognorm = %sysevalf(%sysfunc(sqrt(&scale_lognorm)));

/* Mixing Weights */
%let mix_p1 = 0.7036; /* Weight for Exponential Component */
%let mix_p2 = 0.2964; /* Weight for Lognormal Component */

/* 2. CALCULATE THEORETICAL CDF (The Hard Part) */
data severity_cdf_calc;
    set severity_et7_10k; /* Your dataset name from IMG_3275 */
    
    /* Only look at data above truncation (Safety check) */
    if gross_loss > &L_trunc;

    /* --- COMPONENT 1: EXPONENTIAL --- */
    /* Base CDF at x */
    cdf_exp_x = cdf('EXPONENTIAL', gross_loss, &sigma_truncexp);
    /* Base CDF at Truncation Point (10k) */
    cdf_exp_L = cdf('EXPONENTIAL', &L_trunc, &sigma_truncexp);
    
    /* Truncated CDF Formula: (F(x) - F(L)) / (1 - F(L)) */
    if (1 - cdf_exp_L) > 0 then 
        F_trunc_exp = (cdf_exp_x - cdf_exp_L) / (1 - cdf_exp_L);
    else F_trunc_exp = 0; /* Should not happen if sigma is reasonable */

    /* --- COMPONENT 2: LOGNORMAL --- */
    /* Base CDF at x */
    cdf_logn_x = cdf('LOGNORMAL', gross_loss, &intercept_lognorm, &sdlog_lognorm);
    /* Base CDF at Truncation Point (10k) */
    cdf_logn_L = cdf('LOGNORMAL', &L_trunc, &intercept_lognorm, &sdlog_lognorm);
    
    /* Truncated CDF Formula */
    if (1 - cdf_logn_L) > 0 then
        F_trunc_logn = (cdf_logn_x - cdf_logn_L) / (1 - cdf_logn_L);
    else F_trunc_logn = 0;

    /* --- COMBINE: MIXTURE CDF --- */
    /* F_final = w1*F1 + w2*F2 */
    F_Model = (&mix_p1 * F_trunc_exp) + (&mix_p2 * F_trunc_logn);
    
    keep gross_loss F_Model;
run;

/* 3. SORT DATA (Required for CvM Formula) */
proc sort data=severity_cdf_calc;
    by gross_loss;
run;

/* 4. COMPUTE CONTINUOUS CVM STATISTIC */
/* Note: Using the Continuous formula (Stephens) because Severity is continuous */
data cvm_final;
    set severity_cdf_calc end=last;
    
    /* Get Rank i */
    rank_i = _N_;
    
    /* Get Total N (on first pass) */
    if _N_ = 1 then do;
        dsid = open("severity_cdf_calc");
        N_total = attrn(dsid, "nlobs");
        rc = close(dsid);
    end;
    retain N_total;
    
    /* --- THE CVM FORMULA --- */
    /* Term = ( F(xi) - (2i-1)/2n )^2 */
    term = (F_Model - ( (2*rank_i - 1) / (2*N_total) ))**2;
    
    /* Accumulate Sum */
    retain Sum_Sq 0;
    Sum_Sq = Sum_Sq + term;
    
    if last then do;
        /* Final Calculation: 1/(12n) + Sum */
        W2_Stat = (1 / (12 * N_total)) + Sum_Sq;
        
        put "-----------------------------------------------";
        put " Cramér–von Mises Statistic (Severity): " W2_Stat;
        put "-----------------------------------------------";
        call symputx('CvM_Severity', W2_Stat);
    end;
run;

/* Check Result */
proc print data=cvm_final(obs=1); 
    var W2_Stat; 
run;
