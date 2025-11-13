/* Guard Alpha lower bound against <= 0 safely */
%if %sysevalf(%superq(aL)=,boolean) %then %let aL = 1E-6;
%else %if %sysevalf(&aL <= 0) %then %let aL = 1E-6;
