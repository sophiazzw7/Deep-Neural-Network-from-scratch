/* How many successful bootstrap refits? */
proc sql;
  select count(*) as n_boot from NB_BOOT;
quit;

/* Look at a few bootstrap parameter draws */
proc print data=NB_BOOT(obs=10); run;

/* Look at the raw percentile table */
proc print data=NB_Boot_CIs; run;

/* If NB_Boot_Summary exists, print it; otherwise create it now: */
%macro show_summary;
  %if %sysfunc(exist(NB_Boot_Summary)) %then %do;
    proc print data=NB_Boot_Summary noobs; run;
  %end;
  %else %do;
    data NB_Boot_Summary;
      length Param $12;
      set NB_Boot_CIs;
      Param='mu (mean)';  Estimate=&mu_hat;  LCL=mu_2_5;    UCL=mu_97_5;    output;
      Param='alpha';      Estimate=&alpha;   LCL=alpha_2_5; UCL=alpha_97_5; output;
      Param='r=1/alpha';  Estimate=&r_hat;   LCL=r_2_5;     UCL=r_97_5;     output;
      Param='p';          Estimate=&p_hat;   LCL=p_2_5;     UCL=p_97_5;     output;
      keep Param Estimate LCL UCL;
    run;

    proc print data=NB_Boot_Summary noobs; run;
  %end;
%mend;

%show_summary;
proc sql;
  select count(*) as n_boot from NB_BOOT;
quit;
