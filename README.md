/*===========================================================
  ET7 severity mixture – simulation-based CvM statistic
  (no CDF calls, no integration)
===========================================================*/
libname ops "/sasdata/mrmg2/users/G07267/MOD13638_2025/Output";

/* 1. ET7 data, truncation at 10,000 */
data severity_et7_10k;
    set ops.cleaned_severity_data;
    if strip('Basel Event Type Level 1'n) =
       "ET7 - Execution Delivery and Process Management"
       and gross_loss >= 10000;
run;

/* 2. Mixture parameters (same as your quantile test) */
%let L_trunc           = 10000;
%let intercept_truncexp = 9.4179;
%let sigma_truncexp     = %sysevalf(%sysfunc(exp(&intercept_truncexp.)));

%let intercept_lognorm  = 11.6984;
%let scale_lognorm      = 1.6015;
%let sdlog_lognorm      = %sysevalf(%sysfunc(sqrt(&scale_lognorm.)));

%let mix_p1 = 0.7036;   /* exponential weight  */
%let mix_p2 = 0.2964;   /* lognormal weight    */

/* 3. Sample size of ET7 data */
proc sql noprint;
    select count(*) into :N_obs trimmed
    from severity_et7_10k;
quit;
%put NOTE: N_obs=&N_obs.;

/*===========================================================
  4. Simulate from fitted mixture (model sample)
     - no CDF, only RAND()
===========================================================*/
%let N_sim = 20000;   /* can increase if you want smoother CDF */

data sim_et7;
    call streaminit(12345);
    do k = 1 to &N_sim.;
        u = rand("UNIFORM");

        if u < &mix_p1. then do;
            /* truncated exponential above L_trunc */
            /* draw Exp and shift above L_trunc    */
            z = &L_trunc. + rand("EXPONENTIAL", &sigma_truncexp.);
        end;
        else do;
            /* truncated lognormal via rejection   */
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

/*===========================================================
  5. For each observed ET7 loss x_i, compute model CDF
     F_model(x_i) ≈ proportion of simulated values <= x_i
===========================================================*/

proc sort data=severity_et7_10k;
    by gross_loss;
run;

proc sort data=sim_et7;
    by gross_loss_sim;
run;

/* Use correlated subquery to get F_model(x_i) */
proc sql;
    create table cvm_data as
    select a.gross_loss,
           /* empirical CDF under model */
           (select count(*) from sim_et7 b
             where b.gross_loss_sim <= a.gross_loss) / &N_sim. as F_model
    from severity_et7_10k as a
    order by a.gross_loss;
quit;

/*===========================================================
  6. Compute CvM statistic:
     CvM = 1/(12n) + Σ_i [F_model(x_i) - (2i-1)/(2n)]^2
===========================================================*/

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
    output out=cvm_sum sum=sum_diff2;
run;

data cvm_stat;
    set cvm_sum;
    N   = &N_obs.;
    CvM = (1/(12*N)) + sum_diff2;
run;

proc print data=cvm_stat;
    title "Simulation-based Cramer–von Mises Statistic for ET7 Severity Mixture";
run;
