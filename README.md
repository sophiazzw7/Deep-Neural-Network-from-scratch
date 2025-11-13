/*---- Pull Intercept and Alpha from PROC COUNTREG ODS table ----*/
proc sql noprint;
  /* Intercept */
  select estimate
    into :b0 trimmed
  from nb_parms
  where upcase(Parameter) = 'INTERCEPT';

  /* Alpha/Dispersion row is named _Alpha in COUNTREG; use LIKE to be safe */
  select estimate
    into :alpha trimmed
  from nb_parms
  where upcase(Parameter) like '%ALPHA%';
quit;

%put NOTE: Intercept=&b0  Alpha=&alpha;
