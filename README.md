proc iml;
   /* ---- Read ET7 losses ---- */
   use severity_et7_10k;
      read all var {gross_loss} into x;
   close severity_et7_10k;

   /* Sort losses */
   call sort(x, 1);
   n = nrow(x);

   /* Parameters from your macros */
   L      = &L_trunc.;
   sigmaE = &sigma_truncexp.;         /* exponential scale */
   muLN   = &intercept_lognorm.;      /* lognormal meanlog */
   sdLN   = &sdlog_lognorm.;          /* lognormal sdlog   */
   p1     = &mix_p1.;
   p2     = &mix_p2.;

   /* ----- Truncated exponential CDF at L ----- */
   start FexpTrunc(z, L, sigma);
      F = j(nrow(z), 1, 0);
      idx = loc(z>=L);
      if ncol(idx)>0 then
         F[idx] = 1 - exp(-(z[idx]-L)/sigma);
      return(F);
   finish;

   /* ----- Truncated lognormal CDF at L ----- */
   start FLNTrunc(z, L, mu, sd);
      F = j(nrow(z), 1, 0);
      idx = loc(z>=L);
      if ncol(idx)>0 then do;
         FL = cdf("LOGNORMAL", L, mu, sd);
         F0 = cdf("LOGNORMAL", z[idx], mu, sd);
         F[idx] = (F0 - FL) / (1 - FL);
      end;
      return(F);
   finish;

   /* ----- Mixture CDF at each observed x_i ----- */
   Fexp = FexpTrunc(x, L, sigmaE);
   FLN  = FLNTrunc(x, L, muLN, sdLN);
   Fmodel = p1*Fexp + p2*FLN;

   /* ----- Empirical mid-ranks (2i−1)/(2n) ----- */
   idx = t(1:n);
   Femp_mid = (2*idx - 1) / (2*n);

   /* ----- CvM statistic: 1/(12n) + Σ (Fmodel - Femp)^2 ----- */
   diff  = Fmodel - Femp_mid;
   CvM   = 1/(12*n) + diff`*diff;

   print n CvM[label="CvM_ET7"];
quit;
