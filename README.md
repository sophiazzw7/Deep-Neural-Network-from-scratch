Absolutely — here’s a drop-in SAS snippet that **adds an Anderson–Darling (AD) goodness-of-fit test for a Negative Binomial** to the developer’s pipeline you showed. It reuses your dataset `frequency_et2` and the `PROC COUNTREG` NB fit, then computes an **AD statistic** and a **bootstrap p-value** (so it’s valid for the discrete NB case).

```sas
/*--------------------------------------------------------------------
  NEGATIVE BINOMIAL FIT  (reuses your developer step)
--------------------------------------------------------------------*/
proc countreg data=frequency_et2;
   model frequency = / dist=NEGBIN;
   ods output ParameterEstimates=nb_parms;
quit;

/* Extract Intercept and Alpha, then convert to NB(p, r) parameterization
   used by CDF/RAND in SAS: p = r/(r + mu), r = dispersion = 1/alpha. */
proc sql noprint;
   select estimate into :b0 trimmed from nb_parms where upcase(variable)='INTERCEPT';
   select estimate into :alpha trimmed from nb_parms where upcase(variable)='ALPHA';
quit;

%let mu_hat = %sysevalf(exp(&b0));               /* mean */
%let r_hat  = %sysevalf(1/&alpha);               /* dispersion (number of successes) */
%let p_hat  = %sysevalf(&r_hat / (&r_hat + &mu_hat));  /* success probability */

/*--------------------------------------------------------------------
  ANDERSON–DARLING STATISTIC for NB(p_hat, r_hat) + BOOTSTRAP p-value
--------------------------------------------------------------------*/
proc iml;
   start AD_NB(x, p, r);              /* AD statistic (continuous formula, bootstrap p-value later) */
      call sort(x, 1);
      n = nrow(x);
      F = j(n,1,.);
      do i = 1 to n;
         F[i] = cdf("NEGBINOMIAL", x[i], p, r);
         /* clamp away from 0/1 to avoid log issues */
         if F[i] < 1e-12 then F[i] = 1e-12;
         if F[i] > 1-1e-12 then F[i] = 1-1e-12;
      end;
      idx = T(1:n);
      A2 = -n - (1/n) * sum( (2*idx-1)#( log(F) + log(1 - F[n+1-idx]) ) );
      return (A2);
   finish;

   /* Read observed data */
   use frequency_et2; read all var {frequency} into y; close frequency_et2;
   n   = nrow(y);
   p   = &p_hat;    r = &r_hat;

   /* Observed AD statistic */
   A2_obs = AD_NB(y, p, r);

   /* Parametric bootstrap for p-value */
   B = 1000; call randseed(12345);
   A2_boot = j(B,1,.);
   do b = 1 to B;
      yb = j(n,1,.);
      call randgen(yb, "NEGBINOMIAL", p, r);
      A2_boot[b] = AD_NB(yb, p, r);
   end;

   /* Empirical p-value: proportion of bootstrapped A2 >= observed */
   pval = (sum(A2_boot >= A2_obs) + 1) / (B + 1);

   create AD_NB_Result var {"A2_obs" "pval" "p_hat" "r_hat" "mu_hat" "B"};
   append; close AD_NB_Result;
quit;

/* Pretty print */
proc print data=AD_NB_Result label noobs;
   label A2_obs="AD Statistic (Observed)"
         pval  ="Bootstrap p-value"
         p_hat ="NB p (success prob)"
         r_hat ="NB r (dispersion)"
         mu_hat="Mean (mu)"
         B     ="Bootstrap reps";
run;
```

**What this does**

* Fits **NB** with `PROC COUNTREG` (as in your code) and grabs **Intercept** and **Alpha**.
* Converts to SAS’s NB parameters **p** and **r** used by `CDF`/`RAND`.
* Computes the **Anderson–Darling statistic** against the fitted NB CDF.
* Uses a **parametric bootstrap (B=1000)** to get a **p-value** appropriate for the **discrete** NB case.
* Produces a neat table with the observed AD, p-value, and fitted parameters.

You can reuse the same block for ET2/ET3/ET7 by changing the input dataset or wrapping in a macro.
