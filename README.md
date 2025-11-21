Yes, we can do a **Kuiper test** using only simulation + data steps, no IML and no CDF calls, so you won’t get the integration errors.

I’ll give you **two-sample Kuiper** code: it compares

* sample 1 = **actual ET7 losses**
* sample 2 = **simulated ET7 losses from your fitted mixture**

This is equivalent (in practice) to a GOF test of “does the fitted model generate data like what we observed?”.

---

## 1. Assume you already have ET7 data + mixture params

You already had this in your ET7 programs:

```sas
libname ops "/sasdata/mrmg2/users/G07267/MOD13638_2025/Output";

data severity_et7_10k;
    set ops.cleaned_severity_data;
    if strip('Basel Event Type Level 1'n) =
       "ET7 - Execution Delivery and Process Management"
       and gross_loss >= 10000;
run;

/* mixture parameters (use your actual values) */
%let L_trunc           = 10000;
%let intercept_truncexp = 9.4179;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp.)));

%let intercept_lognorm  = 11.6984;
%let scale_lognorm      = 1.6015;
%let sdlog_lognorm      = %sysevalf(%sysfunc(sqrt(&scale_lognorm.)));

%let mix_p1 = 0.7036;
%let mix_p2 = 0.2964;
```

---

## 2. Simulate from your fitted severity mixture (model sample)

```sas
%let N_model = 20000;   /* size of simulated sample */

data sim_et7;
    call streaminit(12345);
    do k = 1 to &N_model.;
        u = rand("UNIFORM");

        if u < &mix_p1. then do;
            /* truncated exponential above L_trunc */
            z = &L_trunc. + rand("EXPONENTIAL", &sigma_truncexp.);
        end;
        else do;
            /* truncated lognormal via rejection sampling */
            z = &L_trunc. - 1;
            do while (z < &L_trunc.);
                z = rand("LOGNORMAL", &intercept_lognorm., &sdlog_lognorm.);
            end;
        end;

        gross_loss_sim = z;
        output;
    end;
    keep gross_loss_sim;
run;
```

---

## 3. Build combined dataset and compute empirical CDFs

We’ll do a **two-sample Kuiper**:

* combine the two samples,
* walk through sorted values,
* track cumulative proportions for DATA vs MODEL,
* compute `D+`, `D-`, and `V = D+ + D-`.

```sas
/* sample sizes */
proc sql noprint;
    select count(*) into :nData   from severity_et7_10k;
    select count(*) into :nModel  from sim_et7;
quit;

/* combine observed + simulated */
data combined_et7;
    set severity_et7_10k(in=inData  rename=(gross_loss=x))
        sim_et7          (in=inModel rename=(gross_loss_sim=x));
    length sample $6;
    if inData  then sample = "DATA";
    else            sample = "MODEL";
run;

/* sort by loss value */
proc sort data=combined_et7;
    by x;
run;

/* walk through sorted values and compute CDFs for each sample */
data kuiper_terms_et7;
    set combined_et7;
    retain cData cModel 0;
    if sample = "DATA"  then cData  + 1;
    else                     cModel + 1;

    FData  = cData  / &nData.;
    FModel = cModel / &nModel.;
    diff   = FData - FModel;
run;
```

---

## 4. Compute Kuiper statistic (V = D^+ + D^-)

```sas
proc means data=kuiper_terms_et7 noprint;
    var diff;
    output out=kuiper_raw_et7
        max = Dplus
        min = Dmin_raw;
run;

data kuiper_et7;
    set kuiper_raw_et7;
    Dminus = -Dmin_raw;                 /* we stored min(diff), so flip sign */
    V      = Dplus + Dminus;            /* Kuiper statistic */

    /* optional: scaled Kuiper statistic (like KS scaling) */
    Neff   = (&nData. * &nModel.) / (&nData. + &nModel.);
    V_star = (sqrt(Neff) + 0.155 + 0.24/sqrt(Neff)) * V;
run;

proc print data=kuiper_et7;
    title "Two-sample Kuiper Statistic for ET7 Severity (DATA vs MODEL)";
run;
```

---

### How to read the output

You’ll get something like:

| Dplus | Dmin_raw | Dminus | V | V_star |
| ----- | -------- | ------ | - | ------ |

* `V` is your **Kuiper statistic** (0 = perfect match; larger = worse).
* `V_star` is a scaled version you can compare across ET2/ET7, etc.
* There’s no canned p-value in SAS for Kuiper, but you can **treat it like KS**: small V / V_star and visually good QQ/PP plots → acceptable.

If you want, I can also:

* Give a **bootstrap p-value** for this Kuiper statistic (simulate many model samples and recompute), or
* Adapt the same code for ET2 or for your frequency NB model.
