proc iml;
   /**********************************************
    * 1. Read ET2 severity data (>= &L_trunc)
    **********************************************/
   use severity_et2_10k;
      read all var {gross_loss} into x;
   close;

   idx = loc(x >= &L_trunc & x^=.);
   if ncol(idx)=0 then do;
      print "No valid severity observations for CvM.";
      stop;
   end;
   x = x[idx];
   call sort(x, 1);
   n = nrow(x);
   if n <= 1 then do;
      print "Not enough data for CvM.";
      stop;
   end;

   /**********************************************
    * 2. Parameters from your %LETs
    **********************************************/
   Ltrunc = &L_trunc;              /* 10000 */
   thetaE = &sigma_truncexp;       /* exp(intercept_truncexp) */
   muL    = &intercept_lognorm;    /* lognormal mean(log) */
   sigL   = &sdlog_lognorm;        /* lognormal sd(log) */

   w1 = &mix_p1;                   /* trunc exp weight */
   w2 = &mix_p2;                   /* trunc lognormal weight */

   /* Base CDF values at truncation point for each component */
   F0E_L = 1 - exp(-Ltrunc/thetaE);                       /* exp CDF at L */
   F0L_L = cdf("LOGNORMAL", Ltrunc, muL, sigL);           /* lognorm CDF at L */

   /**********************************************
    * 3. Mixture CDF at each ordered x_i
    *    - Both components truncated at Ltrunc
    **********************************************/
   F = j(n,1,.);
   do i = 1 to n;
      xi = x[i];

      /* exponential, truncated at L */
      F0E_x  = 1 - exp(-xi/thetaE);
      F_E_tr = (F0E_x - F0E_L) / (1 - F0E_L);

      /* lognormal, truncated at L */
      F0L_x  = cdf("LOGNORMAL", xi, muL, sigL);
      F_L_tr = (F0L_x - F0L_L) / (1 - F0L_L);

      /* mixture */
      F[i] = w1*F_E_tr + w2*F_L_tr;
   end;

   /**********************************************
    * 4. CvM statistic (continuous case)
    **********************************************/
   idx = T(1:n);
   U   = (2*idx - 1) / (2*n);
   W2  = 1/(12*n) + sum( (F - U)##2 );

   print n W2[label="CvM_W2_ET2_trunc_mixture"];
quit;
