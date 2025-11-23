/*-------------------------------------------------------------*/
/* CvM test for ET2 severity: TruncExpo + LogNormal mixture    */
/*   - Data set: ops.cleaned_severity_data (var = gross_loss)  */
/*   - Mixture params: macros already created from FMM         */
/*     &sigma_for_rand_exponential   = theta (TruncExpo scale) */
/*     &intercept_from_FMM_lognormal = mu (LogNormal meanlog)  */
/*     &scale_from_FMM_lognormal     = sigma (LogNormal sdlog) */
/*     &mixing_probability_model1    = w1 (TruncExpo weight)   */
/*     &mixing_probability_model2    = w2 (LogNormal weight)   */
/*-------------------------------------------------------------*/

proc iml;
   /* 1. Read severity data (filter to ET2 if needed) */
   use ops.cleaned_severity_data;
      read all var {gross_loss} into x;
      /* if you need ET2 only, add a WHERE in the USE/READ step */
   close;

   /* keep positive, nonmissing values */
   idx = loc(x > 0 & x^=.);
   x   = x[idx];

   n = nrow(x);
   if n <= 1 then do;
      print "Not enough data for CvM.";
      stop;
   end;

   call sort(x, 1);   /* x_(1) <= ... <= x_(n) */

   /* 2. Plug in mixture parameters from macros */
   w1     = &mixing_probability_model1;   /* TruncExpo weight */
   w2     = &mixing_probability_model2;   /* LogNormal weight */
   thetaE = &sigma_for_rand_exponential;  /* TruncExpo scale */
   lowerE = 10000;                        /* truncation point used in FMM */
   muL    = &intercept_from_FMM_lognormal; /* LogNormal mean(log) */
   sigL   = &scale_from_FMM_lognormal;     /* LogNormal sd(log) */

   /* 3. Compute model CDF F(x_i) for the mixture at each ordered x_i */
   F = j(n,1,.);
   do i = 1 to n;
      xi = x[i];

      /* CDF for each component */
      F_exp  = cdf("TRUNCEXPO", xi, thetaE, lowerE);   /* trunc expo CDF */
      F_logn = cdf("LOGNORMAL", xi, muL, sigL);        /* lognormal CDF */

      /* Mixture CDF */
      F[i] = w1*F_exp + w2*F_logn;
   end;

   /* 4. Cramér–von Mises statistic (continuous case) */
   idx = T(1:n);
   U   = (2*idx - 1) / (2*n);              /* (2i-1)/(2n) */
   W2  = 1/(12*n) + sum( (F - U)##2 );

   print n W2[label="CvM_W2 for ET2 severity mixture"];
quit;
