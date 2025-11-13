/*--------------------------------------------------------------------
  CRAMÉR–VON MISES (CvM) TEST for NB(p_hat, r_hat)
  Reuses the same parameters (p_hat, r_hat) from your previous step
--------------------------------------------------------------------*/
proc iml;
   start CvM_NB(x, p, r);
      call sort(x,1);
      n = nrow(x);
      F = j(n,1,.);
      do i = 1 to n;
         F[i] = cdf("NEGBINOMIAL", x[i], p, r);
      end;
      /* CvM statistic = (1/(12n)) + Σ [F_i - (2i-1)/(2n)]² */
      i = T(1:n);
      W2 = (1/(12*n)) + sum( (F - ((2*i - 1)/(2*n)))##2 );
      return(W2);
   finish;

   /* Load data (same dataset as AD test) */
   use &ds_in; read all var {frequency} into y; close &ds_in;

   p = &p_hat; r = &r_hat; n = nrow(y);

   /* Observed CvM statistic */
   W2_obs = CvM_NB(y, p, r);

   /* Bootstrap for p-value */
   B = 1000; call randseed(54321);
   W2_boot = j(B,1,.);
   do b = 1 to B;
      yb = j(n,1,.);
      call randgen(yb, "NEGBINOMIAL", p, r);
      W2_boot[b] = CvM_NB(yb, p, r);
   end;

   pval = (sum(W2_boot >= W2_obs) + 1) / (B + 1);

   create CvM_NB_Result var {"W2_obs" "pval" "p" "r" "B"};
   append; close CvM_NB_Result;
quit;

proc print data=CvM_NB_Result label noobs;
   label W2_obs = "CvM Statistic (Observed)"
         pval    = "Bootstrap p-value"
         p       = "NB p (success prob)"
         r       = "NB r (dispersion)"
         B       = "Bootstrap reps";
run;
