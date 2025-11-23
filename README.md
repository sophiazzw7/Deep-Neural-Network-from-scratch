proc iml;
   /*******************************
    * 1. Read and sort severities *
    *******************************/
   use severity_et2_10k;
      read all var {gross_loss} into x;
   close;

   /* keep >= L_trunc and nonmissing */
   idx = loc(x >= &L_trunc & x^=.);
   if ncol(idx)=0 then do;
      print "No valid observations for AD test."; stop;
   end;
   x = x[idx];
   call sort(x, 1);      /* ascending */
   n = nrow(x);
   if n <= 1 then do;
      print "Not enough observations for AD test."; stop;
   end;

   /***************************************
    * 2. Mixture parameters from %LETs    *
    ***************************************/
   Ltrunc = &L_trunc;            /* truncation point */
   thetaE = &sigma_truncexp;     /* exp scale        */
   muL    = &intercept_lognorm;  /* lognormal meanlog*/
   sigL   = &sdlog_lognorm;      /* lognormal sdlog  */
   w1     = &mix_p1;             /* exp weight       */
   w2     = &mix_p2;             /* lognormal weight */

   /* base CDFs at truncation for each component */
   F0E_L = 1 - exp(-Ltrunc/thetaE);             /* exponential CDF at L */
   F0L_L = cdf("LOGNORMAL", Ltrunc, muL, sigL); /* lognormal CDF at L   */

   /***************************************
    * 3. Compute mixture CDF F(x_i)       *
    ***************************************/
   F = j(n,1,.);

   do i = 1 to n;
      xi = x[i];

      /* base CDFs at xi (before truncation) */
      F0E_x = 1 - exp(-xi/thetaE);
      F0L_x = cdf("LOGNORMAL", xi, muL, sigL);

      /* left-truncated CDFs (xi >= Ltrunc) */
      F_E_tr = (F0E_x - F0E_L) / (1 - F0E_L);
      F_L_tr = (F0L_x - F0L_L) / (1 - F0L_L);

      /* mixture CDF */
      F[i] = w1*F_E_tr + w2*F_L_tr;
   end;

   /* clamp to avoid log(0) issues */
   eps = 1e-10;
   idxLow  = loc(F < eps);
   if ncol(idxLow)>0 then F[idxLow] = eps;
   idxHigh = loc(F > 1-eps);
   if ncol(idxHigh)>0 then F[idxHigh] = 1-eps;

   /***************************************
    * 4. Anderson–Darling statistic A^2   *
    *    A^2 = -n - (1/n) Σ (2i-1)[ln F_i
    *                          + ln(1-F_{n+1-i})]
    ***************************************/
   idx = T(1:n);
   F_fwd = F;            /* F_1,...,F_n      */
   F_rev = F[n:1];       /* F_n,...,F_1      */

   sumTerm = sum( (2*idx - 1) # ( log(F_fwd) + log(1 - F_rev) ) );
   A2 = -n - (1/n) * sumTerm;

   print n A2[label="AD_A2_ET2_trunc_mixture"];
quit;
