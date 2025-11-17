Nice, this is a good choice. Let’s build a **P–P plot with simulation bands** that:

* Uses the **developer’s truncated Exponential + Lognormal mixture**
* Transforms losses through the **model CDF** to get “uniformized” values
* Compares those to **Uniform(0,1) order statistics** via simulation bands
* Uses only DATA steps + PROCs (no IML, no fragile math)

You can run this as one program. Adjust paths / macro values if the developer updates parameters.

---

## 0. Build ET2 ≥ 10,000 severity dataset

```sas
libname lee "/sasdata/sasdata15/rqcral/Users/g17183";

/* ET2 - External Fraud, gross_loss >= 10,000 */
data severity_et2_10k;
    set lee.severity;
    if upcase(strip('Basel Event Type Level 1'n)) = "ET2 - EXTERNAL FRAUD"
       and gross_loss >= 10000;
run;

/* Sanity check */
proc sql;
    select count(*) as n_obs,
           min(gross_loss) as min_loss,
           max(gross_loss) as max_loss
    from severity_et2_10k;
quit;
```

---

## 1. Plug in developer’s mixture parameters (no fitting)

```sas
/* Left truncation threshold */
%let L_trunc = 10000;

/* Component 1: Truncated Exponential */
%let intercept_truncexp = 8.8993;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp)));

/* Component 2: Lognormal */
%let intercept_lognorm = 11.0232;    /* meanlog */
%let scale_lognorm     = 0.6533;     /* variance of log */
%let sdlog_lognorm     = %sysevalf(%sysfunc(sqrt(&scale_lognorm)));

/* Mixture weights (from developer) */
%let mix_p1 = 0.7606;   /* TruncExpo weight  */
%let mix_p2 = 0.2394;   /* Lognormal weight  */

/* Number of simulations for bands */
%let nSim = 5000;
```

---

## 2. Compute model CDF F_model(x) for each observed loss

We compute the **truncated mixture CDF** at each `gross_loss` and then sort these values.

```sas
/* Count observations */
proc sql noprint;
    select count(*) into :N_obs trimmed
    from severity_et2_10k;
quit;
%put NOTE: N_obs = &N_obs.;

/* Compute mixture CDF at each loss and “uniformize” via model CDF */
data pp_data;
    set severity_et2_10k(keep=gross_loss);
    retain F0E_L F0L_L;

    /* Precompute truncation CDF values once */
    if _n_ = 1 then do;
        F0E_L = cdf("EXPONENTIAL", &L_trunc, &sigma_truncexp);
        F0L_L = cdf("LOGNORMAL",   &L_trunc, &intercept_lognorm, &sdlog_lognorm);
    end;

    /* Truncated Exponential component CDF at gross_loss */
    F0E   = cdf("EXPONENTIAL", gross_loss, &sigma_truncexp);
    FexpT = (F0E - F0E_L) / (1 - F0E_L);

    /* Truncated Lognormal component CDF at gross_loss */
    F0L   = cdf("LOGNORMAL", gross_loss, &intercept_lognorm, &sdlog_lognorm);
    FlogT = (F0L - F0L_L) / (1 - F0L_L);

    /* Mixture CDF */
    F_model = &mix_p1*FexpT + &mix_p2*FlogT;

    /* Clamp for numerical safety */
    if F_model < 1e-12 then F_model = 1e-12;
    if F_model > 1-1e-12 then F_model = 1-1e-12;
run;

/* Sort by model CDF and assign empirical ranks */
proc sort data=pp_data; by F_model; run;

data pp_data;
    set pp_data;
    by F_model;
    retain rank 0;
    rank + 1;
    /* empirical CDF value for each ordered point */
    u_emp = rank / (&N_obs + 1);
run;

/* These are the observed P–P points:
   x = u_emp,  y = F_model (sorted) */
proc print data=pp_data(obs=10); run;
```

---

## 3. Simulate Uniform(0,1) order statistics for bands

Under the correct model, `F_model(gross_loss)` should behave like **order statistics of Uniform(0,1)**. So we simulate that directly (no mixture simulation needed).

```sas
/* Simulate Uniform(0,1) samples and sort to get order statistics */
data pp_sim;
    call streaminit(12345);
    do sim_id = 1 to &nSim;
        do i = 1 to &N_obs;
            u = rand("UNIFORM");
            output;
        end;
    end;
    drop i;
run;

/* Sort each simulation and assign rank within that simulation */
proc sort data=pp_sim; by sim_id u; run;

data pp_sim_sorted;
    set pp_sim;
    by sim_id u;
    retain rank;
    if first.sim_id then rank = 0;
    rank + 1;       /* 1..N_obs */
run;

/* Now we want, for each rank, the distribution of u across simulations.
   That gives us the band for F_model(X_(rank)). */

proc sort data=pp_sim_sorted; by rank; run;

/* For each rank, get 2.5%, 50%, 97.5% percentiles of u */
proc univariate data=pp_sim_sorted noprint;
    by rank;
    var u;
    output out=pp_bands
        pctlpts = 2.5 50 97.5
        pctlpre = B_
        pctlname = P2 P50 P97;
run;

/* Merge bands onto observed P–P points */
data pp_plot;
    merge pp_data
          pp_bands;
    by rank;
    /* Rename band variables for clarity */
    B2   = B_P2;
    Bmed = B_P50;
    B97  = B_P97;
run;

proc print data=pp_plot(obs=10); run;
```

---

## 4. Plot the P–P curve with bands

You now have, for each rank:

* `u_emp`  – empirical CDF value (x-axis)
* `F_model` – model CDF at ordered losses (y-axis)
* `B2`, `Bmed`, `B97` – simulation bands for ideal Uniform order stats

You can plot them with `PROC SGPLOT`:

```sas
ods graphics on;

proc sgplot data=pp_plot;
    /* 95% simulation band */
    series x=u_emp y=B2   / lineattrs=(pattern=shortdash) legendlabel="Lower band (2.5%)";
    series x=u_emp y=B97  / lineattrs=(pattern=shortdash) legendlabel="Upper band (97.5%)";

    /* Median Uniform order statistic (should lie on 45° line) */
    series x=u_emp y=Bmed / lineattrs=(pattern=solid) legendlabel="Median under model";

    /* Observed P–P points */
    scatter x=u_emp y=F_model / markerattrs=(symbol=circlefilled size=3)
           legendlabel="Observed P–P";

    /* 45° reference line */
    lineparm x=0 y=0 slope=1 / lineattrs=(pattern=dot) legendlabel="Perfect fit";

    xaxis label="Empirical cumulative probability (u)" min=0 max=1;
    yaxis label="Model CDF at observed losses, F_model(x)" min=0 max=1;
run;

ods graphics off;
```

---

## How to read this plot

* If the model fits well, the **Observed P–P points (dots)** should lie near the **45° line** and mostly **within the band [B2, B97]**.
* Systematic deviations (curving away from the line, or large sections outside the band) indicate misfit:

  * Points **below** the line → model overestimates probabilities (underestimates losses).
  * Points **above** the line → model underestimates probabilities (overestimates losses).
  * Deviations near **u → 1** highlight **tail misfit**, which is key for operational risk.

This is easily explainable to the developer:

> “We mapped the observed ET2 losses through your fitted severity CDF and compared the resulting empirical cumulative probabilities to simulation bands from the ideal Uniform(0,1) distribution. Large, systematic deviations of the P–P curve from the 45° line and outside the 95% band indicate that the mixture does not fully capture the empirical severity distribution.”

---

If you run this and send me a quick description of how the points sit relative to the band (e.g., “they are mostly below the band for u>0.8”), I can help you write a short, precise paragraph describing the result for your validation report and for developer feedback.
