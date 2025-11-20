proc freq data=sim; tables freq_sim / out=emp;
run;

data theo;
   do x = 0 to 2000;
      p = pdf("NEGBINOMIAL", x, p_hat, r_hat);
      output;
   end;
run;

proc compare base=theo compare=emp;
run;
