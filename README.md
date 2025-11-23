proc iml;
   /*******************************
    * 1. Read and sort severities *
    *******************************/
   use severity_et2_10k;
      read all var {gross_loss} into x;
   close;

   idx = loc(x >= &L_trunc & x^=.);
   if ncol(idx)=0 then do;
      print "No valid observations for AD test."; stop;
   end;
   x = x[idx];
   call sort(x, 1);
   n = nrow(x);
   if n <= 1 then do;
      print "Not enough observations for AD test."; stop;
   end;

   /***************************************
    * 2. Mixture parameters from %LETs    *
    ***************************************/
   Ltrunc = &L_trunc;            /* 10000 */
   thetaE = &sigma_truncexp;     /* exp scale        */
   muL    = &intercept_lognorm;  /* lognormal mean   */
   sigL   = &sdlog_lognorm;      /* lognormal sd     */
   w1     = &mix_p1;             /* exp weight       */
   w2     = &mix_p2;             /* lognormal weight */

   /* base CDFs at truncation point */
   F0E_L = 1 - exp(-Ltrunc/thetaE);
   F0L_L = cdf("LOGNORMAL", Ltrunc, muL, sigL);

   /* if numeric integration produced missing, push to safe values */
   eps = 1e-12;
   if F0L_L = . then F0L_L = eps;          /* extremely small prob at L */

   /***************************************
    * 3. Compute mixture CDF F(x_i)       *
    ***************************************/
   F = j(n,1,.);

   do i = 1 to n;
      xi = x[i];

      /* base CDFs before truncation */
      F0E_x = 1 - exp(-xi/thetaE);
      F0L_x = cdf("LOGNORMAL", xi, muL, sigL);

      /* catch integration failures / out-of-range values */
      if F0E_x = . then F0E_x = (xi>0);        /* 0 or 1 depending on sign */
      if F0L_x = . then F0L_x = 1 - eps;       /* treat as almost 1 */

      if F0E_x < eps then F0E_x = eps;
      if F0E_x > 1-eps then F0E_x = 1-eps;
      if F0L_x < eps then F0L_x = eps;
      if F0L_x > 1-eps then F0L_x = 1-eps;

      /* left-truncated CDFs (xi >= Ltrunc) */
      F_E_tr = (F0E_x - F0E_L) / (1 - F0E_L);
      F_L_tr = (F0L_x - F0L_L) / (1 - F0L_L);

      /* clamp truncated CDFs too */
      if F_E_tr < eps then F_E_tr = eps;
      if F_E_tr > 1-eps then F_E_tr = 1-eps;
      if F_L_tr < eps then F_L_tr = eps;
      if F_L_tr > 1-eps then F_L_tr = 1-eps;

      /* mixture CDF */
      F[i] = w1*F_E_tr + w2*F_L_tr;
   end;

   /* final clamp for mixture – avoid log(0) and log(1) */
   idxLow  = loc(F < eps);
   if ncol(idxLow)>0 then F[idxLow] = eps;
   idxHigh = loc(F > 1-eps);
   if ncol(idxHigh)>0 then F[idxHigh] = 1-eps;

   /***************************************
    * 4. Anderson–Darling statistic A^2   *
    ***************************************/
   idx = T(1:n);
   F_fwd = F;          /* F_1,...,F_n      */
   F_rev = F[n:1];     /* F_n,...,F_1      */

   sumTerm = sum( (2*idx - 1) # ( log(F_fwd) + log(1 - F_rev) ) );
   A2 = -n - (1/n) * sumTerm;

   print n A2[label="AD_A2_ET2_trunc_mixture"];
quit;
