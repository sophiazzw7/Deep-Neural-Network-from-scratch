Yes, that’s exactly the right idea – what you’re describing is a **parametric bootstrap on the quantiles**:

* For each replication, simulate a full sample with the **same n as the original data** from the fitted model.
* Compute Q50, Q75, Q90, Q95, Q99 for that simulated sample.
* After 100 (or 1,000) replications, you have a distribution of Q50, Q75, … under the model.
* Then you check whether the **empirical** Q50, Q75, … fall inside, say, the 2.5–97.5% range of those simulated quantiles.

Statistically that’s very clean, and it’s exactly what we were trying to do — we’ll just implement it in the way you described.

Below is a **self-contained SAS program** for ET2 frequency using the NB model from `PROC COUNTREG`. You can change `nSim` or reuse it for other ETs by swapping the dataset/variable.

---

### 1. Fit NB and get parameters (if not already done)

```sas
/* Fit NB frequency model */
proc countreg data=ops.ef_dataset;
    model frequency = / dist=negbin;
    ods output ParameterEstimates = nb_parms;
quit;

/* Extract intercept and alpha */
proc sql noprint;
    select estimate into :b0 trimmed
    from nb_parms
    where upcase(parameter)='INTERCEPT';

    select estimate into :alpha trimmed
    from nb_parms
    where upcase(parameter) like '%ALPHA%';
quit;

/* Convert COUNTREG params to RAND('NEGBINOMIAL', p, r) params */
%let mu_hat = %sysevalf(%sysfunc(exp(&b0)));
%let r_hat  = %sysevalf(1/&alpha);
%let p_hat  = %sysevalf(&r_hat/(&r_hat + &mu_hat));
%put NOTE: mu_hat=&mu_hat r_hat=&r_hat p_hat=&p_hat;
```

---

### 2. Get original sample size and empirical quantiles

```sas
/* Sample size */
proc sql noprint;
    select count(*) into :nObs trimmed
    from ops.ef_dataset;
quit;
%put NOTE: nObs=&nObs;

/* Empirical quantiles */
proc univariate data=ops.ef_dataset noprint;
    var frequency;
    output out=emp_freq
        pctlpts = 50 75 90 95 99
        pctlpre = Q_;
run;
```

---

### 3. Parametric bootstrap: simulate nObs obs per replication, repeat 100 times

```sas
%let nSim = 100;   /* you can increase to e.g. 1000 later */

data sim_freq;
    call streaminit(123456);
    do sim_id = 1 to &nSim;
        do i = 1 to &nObs;          /* same n as original data */
            freq_sim = rand("NEGBINOMIAL", &p_hat, &r_hat);
            output;
        end;
    end;
run;
```

---

### 4. Compute quantiles for each simulated dataset (one row per sim_id)

```sas
proc univariate data=sim_freq noprint;
    by sim_id;
    var freq_sim;
    output out=sim_each
        pctlpts = 50 75 90 95 99
        pctlpre = P_;
run;
```

---

### 5. Get 2.5%, 50%, 97.5% across simulations for each quantile

(and you can change `2.5 97.5` to `5 95` etc if you want a different band)

```sas
/* Q50 band */
proc univariate data=sim_each noprint;
    var P_50;
    output out=band50
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q50 P50_Q50 P97_Q50;
run;

/* Q75 band */
proc univariate data=sim_each noprint;
    var P_75;
    output out=band75
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q75 P50_Q75 P97_Q75;
run;

/* Q90 band */
proc univariate data=sim_each noprint;
    var P_90;
    output out=band90
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q90 P50_Q90 P97_Q90;
run;

/* Q95 band */
proc univariate data=sim_each noprint;
    var P_95;
    output out=band95
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q95 P50_Q95 P97_Q95;
run;

/* Q99 band */
proc univariate data=sim_each noprint;
    var P_99;
    output out=band99
        pctlpts  = 2.5 50 97.5
        pctlname = P2_Q99 P50_Q99 P97_Q99;
run;

/* Combine bands */
data sim_band;
    merge band50 band75 band90 band95 band99;
run;
```

---

### 6. Final comparison: does the empirical quantile fall in the band?

```sas
data freq_quantile_test;
    merge emp_freq sim_band;   /* both single-row */
    /* optional indicators: 1 = empirical is inside 95% band */
    in_band_Q50 = (Q_50 between P2_Q50 and P97_Q50);
    in_band_Q75 = (Q_75 between P2_Q75 and P97_Q75);
    in_band_Q90 = (Q_90 between P2_Q90 and P97_Q90);
    in_band_Q95 = (Q_95 between P2_Q95 and P97_Q95);
    in_band_Q99 = (Q_99 between P2_Q99 and P97_Q99);
run;

proc print data=freq_quantile_test;
run;
```

The printed table will show, for each quantile:

* `Q_50` etc: empirical quantile from the real data
* `P2_Q50`, `P50_Q50`, `P97_Q50`: lower/median/upper of the simulated quantiles
* `in_band_Q50` etc: 1 if empirical is inside the 95% range, 0 otherwise

That matches exactly the procedure you described:

> “simulate n observations = original data size, repeat 100 times, get a distribution of Q50/Q75/…, then check if the original Q50/Q75/… fall in that range.”

If you run this and share the output, I can help you interpret how it lines up with the AD test.
