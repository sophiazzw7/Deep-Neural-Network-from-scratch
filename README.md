/*-----------------------------------------------------------
  1. Get sample size of ET7 data
-----------------------------------------------------------*/
proc sql noprint;
    select count(*) into :N_obs trimmed
    from severity_et7_10k;
quit;
%put NOTE: N_obs = &N_obs.;

/*-----------------------------------------------------------
  2. Simulate from the fitted mixture (model sample)
     - only RAND(), no CDF calls
-----------------------------------------------------------*/
%let N_model = 20000;   /* model sample size for CDF approx */

data sim_model;
    call streaminit(12345);
    do k = 1 to &N_model.;
        u = rand("UNIFORM");

        if u < &mix_p1. then do;
            /* truncated exponential above L_trunc */
            z = &L_trunc. + rand("EXPONENTIAL", &sigma_truncexp.);
        end;
        else do;
            /* truncated lognormal via rejection sampling */
            z = &L_trunc. - 1;
            do while (z < &L_trunc.);
                z = rand("LOGNORMAL", &intercept_lognorm., &sdlog_lognorm.);
            end;
        end;

        gross_loss_sim = z;
        output;
    end;
    keep gross_loss_sim;
run;

/*-----------------------------------------------------------
  3. For each observed loss x, approximate F_model(x)
     as P_model(X <= x) using the simulated sample
-----------------------------------------------------------*/
proc sort data=severity_et7_10k;
    by gross_loss;
run;

proc sort data=sim_model;
    by gross_loss_sim;
run;

proc sql;
    create table cvm_data as
    select a.gross_loss,
           /* empirical CDF under the model via simulation */
           (select count(*) from sim_model b
             where b.gross_loss_sim <= a.gross_loss)
           / &N_model. as F_model
    from severity_et7_10k as a
    order by a.gross_loss;
quit;

/*-----------------------------------------------------------
  4. Compute CvM statistic:
     CvM = 1/(12n) + Σ_i [F_model(x_i) - (2i-1)/(2n)]^2
-----------------------------------------------------------*/
data cvm_terms;
    set cvm_data;
    retain i;
    if _N_ = 1 then i = 0;
    i + 1;

    n = &N_obs.;
    F_emp_mid = (2*i - 1) / (2*n);
    diff  = F_model - F_emp_mid;
    diff2 = diff*diff;
run;

proc means data=cvm_terms noprint;
    var diff2;
    output out=cvm_sum sum = sum_diff2;
run;

data cvm_stat_et7;
    set cvm_sum;
    N_obs = &N_obs.;
    CvM   = (1/(12*N_obs)) + sum_diff2;
run;

proc print data=cvm_stat_et7;
    title "Simulation-based Cramer–von Mises Statistic for ET7 Severity Mixture";
run;
