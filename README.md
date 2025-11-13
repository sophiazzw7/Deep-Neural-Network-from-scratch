data _null_;
  aL = input(symget('aL'), best.);
  if missing(aL) or aL <= 0 then call symputx('aL', 1E-6);
run;
