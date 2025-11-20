proc iml;
/* Fill with your NB parameters */
mu  = 225;       /* example */
k   = 1.5;

/* Convert to NB parameters SAS uses */
p = k/(k+mu); /* success probability */

/* Percentile breaks */
pct = do(0,1,0.1);
q   = quantile("negbinomial", pct, k, p);

print pct q;
quit;
