/*------------------------------------------------------------
  0. Setup & ET7 severity data
------------------------------------------------------------*/
libname ops "/sasdata/mrmg2/users/G07267/MOD13638_2025/Output";

data severity_et7_10k;
    set ops.cleaned_severity_data;
    if strip('Basel Event Type Level 1'n) = "ET7 - Execution Delivery and Process Management"
       and gross_loss >= 10000;
run;

/* Lower truncation level */
%let L_trunc = 10000;

/*------------------------------------------------------------
  1. Mixture parameters (from model documentation)
     - Component 1: truncated exponential
     - Component 2: truncated lognormal
------------------------------------------------------------*/
%let intercept_truncexp = 9.4179;      /* log-mean for exp scale, given by dev */
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp.)));

%let intercept_lognorm  = 11.6984;     /* mu for underlying lognormal */
%let scale_lognorm      = 1.6015;      /* variance on log scale */
%let sdlog_lognorm      = %sysevalf(%sysfunc(sqrt(&scale_lognorm.)));

%let mix_p1 = 0.7036;                  /* weight for exp component */
%let mix_p2 = 0.2964;                  /* weight for lognormal component */

/* Sample size */
proc sql noprint;
    select count(*) into :N_obs trimmed from severity_et7_10k;
quit;
%put NOTE: N_obs=&N_obs.;

/*------------------------------------------------------------
  2. Helper: mixture CDF for a value x >= L_trunc
     (truncated exponential + truncated lognormal)
------------------------------------------------------------*/
/* We’ll compute it directly in data steps via the same formula
   to avoid macro-function complications. */

/*------------------------------------------------------------
  3. KS statistic for actual ET7 data
------------------------------------------------------------*/
proc sort data=severity_et7_10k; 
    by gross_loss; 
run;

data ks_actual;
    set severity_et7_10k;
    by gross_loss;
    retain n;
    if _N_ = 1 then n = 0;
    n + 1;

    /* Empirical CDF */
    F_emp = n / &N_obs.;

    /* Mixture model CDF evaluated at gross_loss */
    if gross_loss < &L_trunc. then do;
        F_exp_trunc  = 0;
        F_logn_trunc = 0;
    end;
    else do;
        /* Truncated exponential above L_trunc is L + Exp(sigma) */
        F_exp_trunc = cdf('EXPONENTIAL', gross_loss - &L_trunc., &sigma_truncexp.);

        /* Truncated lognormal at L_trunc */
        FL  = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0  = cdf('LOGNORMAL', gross_loss, &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn_trunc = 0;
        else F_logn_trunc = (F0 - FL) / (1 - FL);
    end;

    F_model = &mix_p1. * F_exp_trunc + &mix_p2. * F_logn_trunc;

    KS_abs = abs(F_emp - F_model);
run;

proc sql noprint;
    select max(KS_abs) into :KS_real from ks_actual;
quit;
%put NOTE: Real KS statistic for ET7 severity = &KS_real.;

/*------------------------------------------------------------
  4. Parametric bootstrap: simulate from mixture model
------------------------------------------------------------*/
%let nSim = 10000;

data sim_sev;
    call streaminit(12345);

    do sim_id = 1 to &nSim.;
        do i = 1 to &N_obs.;
            u = rand("UNIFORM");

            /* Component 1: truncated exponential (via rejection) */
            if u < &mix_p1. then do;
                z = &L_trunc. - 1;
                do while (z < &L_trunc.);
                    z = rand("EXPONENTIAL", &sigma_truncexp.);
                end;
            end;

            /* Component 2: truncated lognormal (via rejection) */
            else do;
                z = &L_trunc. - 1;
                do while (z < &L_trunc.);
                    z = rand("LOGNORMAL", &intercept_lognorm., &sdlog_lognorm.);
                end;
            end;

            gross_loss_sim = z;
            output;
        end;
    end;

    drop i u z;
run;

/*------------------------------------------------------------
  5. KS statistic for each bootstrap sample
------------------------------------------------------------*/
proc sort data=sim_sev;
    by sim_id gross_loss_sim;
run;

data ks_boot_raw;
    set sim_sev;
    by sim_id gross_loss_sim;

    retain n;
    if first.sim_id then n = 0;
    n + 1;

    /* Empirical CDF in this bootstrap sample */
    F_emp = n / &N_obs.;

    /* Mixture model CDF at gross_loss_sim */
    if gross_loss_sim < &L_trunc. then do;
        F_exp_trunc  = 0;
        F_logn_trunc = 0;
    end;
    else do;
        F_exp_trunc = cdf('EXPONENTIAL', gross_loss_sim - &L_trunc., &sigma_truncexp.);

        FL  = cdf('LOGNORMAL', &L_trunc., &intercept_lognorm., &sdlog_lognorm.);
        F0  = cdf('LOGNORMAL', gross_loss_sim, &intercept_lognorm., &sdlog_lognorm.);
        if FL >= 0.999999 then F_logn_trunc = 0;
        else F_logn_trunc = (F0 - FL) / (1 - FL);
    end;

    F_model = &mix_p1. * F_exp_trunc + &mix_p2. * F_logn_trunc;

    KS_abs = abs(F_emp - F_model);
run;

/* Collapse to one KS statistic per sim_id */
proc sql;
    create table ks_boot as
    select sim_id,
           max(KS_abs) as KS_boot
    from ks_boot_raw
    group by sim_id;
quit;

/*------------------------------------------------------------
  6. Bootstrap summary & p-value
------------------------------------------------------------*/
proc means data=ks_boot n mean std p5 p50 p95;
    var KS_boot;
run;

proc sql noprint;
    select (sum(KS_boot >= &KS_real.) + 1) / (count(*) + 1)
    into :p_boot
    from ks_boot;
quit;

%put NOTE: Bootstrap KS p-value for ET7 severity = &p_boot.;
