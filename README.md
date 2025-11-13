/* ---- Robust Cramér–von Mises (CvM) for NB with bootstrap p-value ---- */
%put NOTE: p_hat=&p_hat r_hat=&r_hat ds=&ds_in;

/* Quick sanity checks (optional but helpful) */
proc means data=&ds_in n nmiss min max; var frequency; run;

proc iml;
   /* Safe CDF for NB with clamping */
   start SafeCDF_NB(x, p, r);
      n = nrow(x);
      F = j(n,1,.);
      do i = 1 to n;
         Fi = cdf("NEGBINOMIAL", x[i], p, r);
         if Fi < 1e-12 then Fi = 1e-12;        /* avoid 0/1 edge cases */
         if Fi > 1-1e-12 then Fi = 1-1e-12;
         F[i] = Fi;
      end;
      return(F);
   finish;

   /* CvM statistic: (1/(12n)) + Σ (F_i - (2i-1)/(2n))^2 */
   start CvM_NB(x, p, r);
      call sort(x,1);
      n  = nrow(x);
      F  = SafeCDF_NB(x, p, r);
      i  = T(1:n);
      W2 = (1/(12*n)) + sum( (F - ((2*i - 1)/(2*n)))##2 );
      return(W2);
   finish;

   /* --- Read data --- */
   use &ds_in; read all var {frequency} into y; close &ds_in;

   /* Validate inputs */
   p = &p_hat; r = &r_hat;
   if ^(p>0 & p<1 & r>0) then do;
      msg = "ERROR: Invalid NB parameters: p=" + char(p) + ", r=" + char(r);
      print msg; stop;
   end;
   if any(y<0) | any(y ^= round(y)) then do;
      print "ERROR: 'frequency' must be nonnegative integers."; stop;
   end;

   /* Observed statistic */
   W2_obs = CvM_NB(y, p, r);

   /* Bootstrap p-value */
   n = nrow(y); B = 1000; call randseed(54321);
   W2_boot = j(B,1,.);
   do b = 1 to B;
      yb = j(n,1,.);
      call randgen(yb, "NEGBINOMIAL", p, r);
      W2_boot[b] = CvM_NB(yb, p, r);
   end;
   pval = (sum(W2_boot >= W2_obs) + 1) / (B + 1);

   create CvM_NB_Result var {"W2_obs" "pval" "p" "r" "B"}; append; close;
quit;

proc print data=CvM_NB_Result label noobs;
  label W2_obs="CvM Statistic (Observed)"
        pval  ="Bootstrap p-value"
        p     ="NB p (success prob)"
        r     ="NB r (dispersion)"
        B     ="Bootstrap reps";
run;
