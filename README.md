Yes, we can reuse the same “parametric bootstrap quantile test” idea you had for severity, but now for the **ET2 frequency negative binomial**.

Below is **full SAS code** you can paste **after** the block where you’ve already fit the NB and created:

* `freq_et2` (only `frequency`)
* macro vars: `mu_hat`, `r_hat`, `p_hat`, `N_et2` (from your screenshot)

---

### 🔹 1. Empirical quantiles of ET2 frequency

```sas
/* STEP A: Empirical quantiles from observed ET2 frequency */
proc univariate data=freq_et2 noprint;
    var frequency;
    output out=obs_q
        pctlpts = 50 75 90 95 99
        pctlpre = Q_;
run;

/* (Optional) check observed quantiles */
proc print data=obs_q;
    title "Observed ET2 Frequency Quantiles";
run;
```

---

### 🔹 2. Simulate from fitted NB and compute quantiles per simulation

```sas
/* STEP B: Parametric bootstrap from fitted NB(freq) */

%let nSim = 10000;   /* number of bootstrap simulations */

/* Simulate N_et2 counts from NB(mu_hat, r_hat) in each simulation */
data sim_freq;
    call streaminit(12345);
    do sim_id = 1 to &nSim.;
        do i = 1 to &N_et2.;
            /* RAND('NEGBINOMIAL', p, r) : r = size, p = success prob */
            freq_sim = rand("NEGBINOMIAL", &p_hat., &r_hat.);
            output;
        end;
    end;
    drop i;
run;

/* For each simulation, get sample quantiles of freq_sim */
proc sort data=sim_freq;
    by sim_id;
run;

proc univariate data=sim_freq noprint;
    by sim_id;
    var freq_sim;
    output out=sim_q
        pctlpts = 50 75 90 95 99
        pctlpre = Q_;
run;
```

---

### 🔹 3. Get 2.5%–97.5% bands for each quantile across simulations

```sas
/* STEP C: For each quantile (50,75,90,95,99), get 2.5 and 97.5 percentiles
           across the nSim simulations (bootstrap CI bands) */

proc univariate data=sim_q noprint;
    var Q_50;
    output out=b_Q50
        pctlpts = 2.5 50 97.5
        pctlpre = B50_
        pctlname = P2 P50 P97;
run;

proc univariate data=sim_q noprint;
    var Q_75;
    output out=b_Q75
        pctlpts = 2.5 50 97.5
        pctlpre = B75_
        pctlname = P2 P50 P97;
run;

proc univariate data=sim_q noprint;
    var Q_90;
    output out=b_Q90
        pctlpts = 2.5 50 97.5
        pctlpre = B90_
        pctlname = P2 P50 P97;
run;

proc univariate data=sim_q noprint;
    var Q_95;
    output out=b_Q95
        pctlpts = 2.5 50 97.5
        pctlpre = B95_
        pctlname = P2 P50 P97;
run;

proc univariate data=sim_q noprint;
    var Q_99;
    output out=b_Q99
        pctlpts = 2.5 50 97.5
        pctlpre = B99_
        pctlname = P2 P50 P97;
run;

/* Merge all bands into one row */
data sim_bands;
    merge b_Q50 b_Q75 b_Q90 b_Q95 b_Q99;
run;
```

---

### 🔹 4. Final “quantile test” table: actual vs bootstrap bands

```sas
/* STEP D: Combine observed quantiles with bootstrap bands */
data quantile_test_et2;
    if _N_ = 1 then set obs_q;
    set sim_bands;
run;

proc print data=quantile_test_et2 noobs;
    title "ET2 Frequency – Quantile Test via NB Parametric Bootstrap";
    var Q_50  B50_P2  B50_P97
        Q_75  B75_P2  B75_P97
        Q_90  B90_P2  B90_P97
        Q_95  B95_P2  B95_P97
        Q_99  B99_P2  B99_P97;
run;
```

---

This gives you a **single table** with:

* Empirical ET2 frequency quantiles (`Q_50`, `Q_75`, `Q_90`, `Q_95`, `Q_99`)
* The **2.5% and 97.5% model-based bands** for each quantile (`Bxx_P2`, `Bxx_P97`)

Then you can:

* Check if each `Q_xx` lies inside its `[P2, P97]` band
* Be more forgiving for Q90–Q99 (just interpret as diagnostic / instability, not hard fail)

If you want, next I can add **pass/fail flags** with the forgiving rules we discussed (strict for 50–75, one-sided for 90–95, diagnostic only for 99).
