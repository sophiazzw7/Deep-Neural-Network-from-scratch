/* 1) Inspect types if needed */
proc contents data=ops.operational_risk_loss_forecast varnum; run;

/* 2) Create numeric version(s) of any money fields that are character */
data work.orlf_num;
  set ops.operational_risk_loss_forecast;

  /* Recovery Amount: character -> numeric */
  length recovery_amount_num 8;
  length _raw $200;

  _raw = strip('Recovery Amount'n);

  /* Detect parentheses = negative */
  length _neg 3;
  _neg = (index(_raw,'(') > 0);

  /* Remove $, commas, spaces, and parentheses */
  _raw = compress(_raw, ' $,()');

  /* Convert to numeric; if blank, leave as missing . */
  if not missing(_raw) then recovery_amount_num = input(_raw, best32.);
  if _neg and not missing(recovery_amount_num) then recovery_amount_num = -recovery_amount_num;

  drop _raw _neg;
run;

/* 3) Now run PROC MEANS on numeric fields */
proc means data=work.orlf_num n nmiss min p1 p5 p25 median p75 p95 p99 max mean std;
  var 'GL Amount'n recovery_amount_num 'Gross Loss Amount'n;
run;
