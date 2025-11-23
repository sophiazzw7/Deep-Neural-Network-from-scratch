proc iml;
   /**********************************************
    * 1. Read ET2 severity data (>= &L_trunc)
    **********************************************/
   use severity_et2_10k;
      read all var {gross_loss} into x;
   close;

   /* keep positive, nonmissing values at/above truncation */
   idx = loc(x >= &L_trunc & x^=.);
   if ncol(idx)=0 then do;
      print "No valid severity observations for CvM.";
      stop;
   end;
   x = x[idx];

   /* sort ascending for CvM computation */
   call sort(x, 1);
   n = nrow(x);
   if n <= 1 then do;
      print "Not enough data for CvM.";
      stop;
   end;

   /**********************************************
    * 2. Plug in mixture parameters from macros
    **********************************************/
   Ltrunc = &L_trunc;              /* truncation point */
   thetaE = &sigma_truncexp;       /* exponential scale (pre-truncation) */
   muL    = &intercept_lognorm;    /* lognormal mean(log) */
   sigL   = &sdlog_lognorm;        /* lognormal sd(log) */

   w1 = &mix_p1;                   /* weight: truncated exponential */
   w2 = &mix_p2;                   /* weight: lognormal */

   /* precompute F0(L) for base exponential (before truncation) */
   F0L = 1 - exp(-Ltrunc/thetaE);

   /**********************************************
    * 3. Compute model CDF F(x_i) at each ordered x_i
    *    - Truncated exponential CDF:
    *        F_trunc(x) = (F0(x) - F0(L)) / (1 - F0(L)), x >= L
    *      where F0(x) = 1 - exp(-x/thetaE)
    **********************************************/
   F = j(n,1,.);
   do i = 1 to n;
      xi  = x[i];

      /* base exponential CDF */
      F0x = 1 - exp(-xi/thetaE);

      /* left-truncated exponential CDF at Ltrunc */
      F_exp_trunc = (F0x - F0L) / (1 - F0L);

      /* lognormal CDF */
      F_logn = cdf("LOGNORMAL", xi, muL, sigL);

      /* mixture CDF */
      F[i] = w1*F_exp_trunc + w2*F_logn;
   end;

   /**********************************************
    * 4. Cramér–von Mises statistic (continuous case)
    *    W^2 = 1/(12n) + sum_{i=1}^n [F(x_i) - (2i-1)/(2n)]^2
    **********************************************/
   idx = T(1:n);
   U   = (2*idx - 1) / (2*n);      /* (2i-1)/(2n) */

   W2  = 1/(12*n) + sum( (F - U)##2 );

   print n W2[label="CvM_W2_ET2_mixture"];
quit;
