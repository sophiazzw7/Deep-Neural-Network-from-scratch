/* Robust Cramér–von Mises (CvM) with mid-P for discrete NB + bootstrap p-value */
%put NOTE: p_hat=&p_hat r_hat=&r_hat ds=&ds_in;

proc iml;
   /* Mid-distribution transform for discrete NB:
      U = F(x-1) + 0.5*P(X=x)  (clamped to (eps,1-eps)) */
   start MidU_NB(x, p, r);
      eps = 1e-10;
      n   = nrow(x);
      U   = j(n,1,.);
      do i = 1 to n;
         xi   = x[i];
         Fm1  = (xi>0) ? cdf("NEGBINOMIAL", xi-1, p, r) : 0;  /* F(x-) */
         pmf  = pdf("NEGBINOMIAL", xi, p, r);                /* P(X=xi) */
         ui   = Fm1 + 0.5*pmf;
         if ui < eps then ui = eps;
         if ui > 1-eps then ui = 1-eps;
         U[i] = ui;
      end;
      return(U);
   finish;

   /* CvM statistic using mid-P U values */
   start CvM_mid(x, p, r);
      call sort(x,1);
      n  = nrow(x);
      U  = MidU_NB(x, p, r);
      i  = T(1:n);
      W2 = (1/(12*n)) + sum( (U - ((2*i - 1)/(2*n)))##2 );
      return(W2);
   finish;

   /* --- Read data --- */
   use &ds_in; read all var {frequency} into y; close &ds_in;

   /* Validate inputs */
   p = &p_hat; r = &r_hat;
   if ^(p>0 & p<1 & r>0) then do;
      print "ERROR: Invalid NB parameters", p r; stop;
   end;
   if any(y<0) | any(y ^= round(y)) then do;
      print "ERROR: 'frequency' must be nonnegative integers."; stop;
   end;

   /* Observed CvM statistic (mid-P) */
   W2_obs = CvM_mid(y, p, r);

   /* Parametric bootstrap for p-value */
   n = nrow(y); B = 1000; call randseed(98765);
   W2_boot = j(B,1,.);
   do b = 1 to B;
      yb = j(n,1,.);
      call randgen(yb, "NEGBINOMIAL", p, r);
      W2_boot[b] = CvM_mid(yb, p, r);
   end;
   pval = (sum(W2_boot >= W2_obs) + 1) / (B + 1);

   create CvM_NB_Result var {"W2_obs" "pval" "p" "r" "B"}; append; close;
quit;

proc print data=CvM_NB_Result label noobs;
  label W2_obs="CvM Statistic (mid-P, Observed)"
        pval  ="Bootstrap p-value"
        p     ="NB p (success prob)"
        r     ="NB r (dispersion)"
        B     ="Bootstrap reps";
run;
