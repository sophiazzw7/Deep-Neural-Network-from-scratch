/*-----------------------------------------------------------
  STEP 2: Create bins for chi-square test
------------------------------------------------------------*/
data freq_et2_bins;
    set freq_et2;

    length bin $20;

    if frequency <= 2 then bin = '0-2';
    else if frequency <= 4 then bin = '3-4';
    else if frequency <= 6 then bin = '5-6';
    else if frequency <= 9 then bin = '7-9';
    else if frequency <= 12 then bin = '10-12';
    else bin = '13+';
run;

/* Observed bin frequencies */
proc freq data=freq_et2_bins noprint;
    tables bin / out=obs_bins;
run;
/*-----------------------------------------------------------
  STEP 3: Compute expected counts for each bin from NB model
------------------------------------------------------------*/

data exp_bins;
    set obs_bins;

    select (bin);
        when ('0-2') do;
            p_bin = cdf("NEGBINOMIAL",2,&p_hat,&r_hat);
        end;
        when ('3-4') do;
            p_bin =  cdf("NEGBINOMIAL",4,&p_hat,&r_hat)
                   - cdf("NEGBINOMIAL",2,&p_hat,&r_hat);
        end;
        when ('5-6') do;
            p_bin =  cdf("NEGBINOMIAL",6,&p_hat,&r_hat)
                   - cdf("NEGBINOMIAL",4,&p_hat,&r_hat);
        end;
        when ('7-9') do;
            p_bin =  cdf("NEGBINOMIAL",9,&p_hat,&r_hat)
                   - cdf("NEGBINOMIAL",6,&p_hat,&r_hat);
        end;
        when ('10-12') do;
            p_bin =  cdf("NEGBINOMIAL",12,&p_hat,&r_hat)
                   - cdf("NEGBINOMIAL",9,&p_hat,&r_hat);
        end;
        when ('13+') do;
            p_bin =  1 - cdf("NEGBINOMIAL",12,&p_hat,&r_hat);
        end;
        otherwise p_bin = .;
    end;

    Expected = &N_et2 * p_bin;
    rename count = Observed;
run;

proc print data=exp_bins noobs;
    var bin Observed Expected;
run;
/*-----------------------------------------------------------
  STEP 4: Compute chi-square test statistic
------------------------------------------------------------*/
data chi_sq_et2;
    set exp_bins end=eof;
    chisq_term = (Observed - Expected)**2 / Expected;

    retain chisq 0;
    chisq + chisq_term;

    if eof then do;
        df = _n_ - 2 - 1;  /* bins - parameters - 1 */
        p_value = 1 - probchi(chisq, df);
        output;
    end;
run;

proc print data=chi_sq_et2 noobs label;
    label chisq = "Chi-square Statistic"
          df    = "Degrees of Freedom"
          p_value = "p-value";
run;
