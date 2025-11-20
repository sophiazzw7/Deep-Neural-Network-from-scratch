/* Compute the KS statistic for actual data */
proc sort data=severity_et2_10k; by gross_loss; run;

data actual_cdf;
    set severity_et2_10k;
    by gross_loss;
    n + 1;
    cdf_emp = n / _N_;
    cdf_model = <PUT_YOUR_MODEL_CDF_FORMULA_HERE>;
    ks_abs = abs(cdf_emp - cdf_model);
run;

proc sql noprint;
    select max(ks_abs) into :KS_real from actual_cdf;
quit;
%put NOTE: Real KS = &KS_real;
/* KS statistic for each bootstrap replication */
proc sort data=sim_sev; by sim_id gross_loss_sim; run;

data ks_boot_raw;
    set sim_sev;
    by sim_id gross_loss_sim;

    retain n_total;
    if first.sim_id then n_total=0;
    n_total+1;

    cdf_emp = _N_ / n_total;
    cdf_model = <MODEL_CDF(gross_loss_sim)>;

    ks_abs = abs(cdf_emp - cdf_model);
run;

proc sql;
    create table ks_boot as
    select sim_id, max(ks_abs) as KS_boot
    from ks_boot_raw
    group by sim_id;
quit;
proc sql noprint;
    select sum(KS_boot >= &KS_real) / count(*)
    into :pvalue
    from ks_boot;
quit;

%put NOTE: Bootstrap KS p-value = &pvalue;
