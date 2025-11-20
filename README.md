/* Compute historical maximum loss */
proc sql noprint;
    select max(gross_loss) into :hist_max trimmed
    from severity_et7_10k;   /* <-- change dataset here for ET2 */
quit;
/*----------------------------------------------------------
   Simulate bootstrap datasets with horizon-adjusted tail
-----------------------------------------------------------*/

%let nSim = 10000;    /* number of bootstrap runs */

/* N_obs = size of observed dataset */
proc sql noprint;
    select count(*) into :N_obs trimmed
    from severity_et7_10k;   /* <-- update for ET2/ET7 */
quit;

data sim_sev;
    call streaminit(12345);
    do sim_id = 1 to &nSim;
        do i = 1 to &N_obs;

            u = rand("UNIFORM");

            /* Component 1: truncated exponential */
            if u < &mix_p1 then do;
                z = &L_trunc - 1;
                do while (z < &L_trunc);
                    z = rand("EXPONENTIAL", &sigma_truncexp);
                end;
            end;

            /* Component 2: truncated lognormal */
            else do;
                z = &L_trunc - 1;
                do while (z < &L_trunc);
                    z = rand("LOGNORMAL", &intercept_lognorm, &sdlog_lognorm);
                end;
            end;

            /*-------------------------------------------
                Horizon-adjustment: CAP the tail
                If simulated loss > historical maximum,
                set z = hist_max.
            --------------------------------------------*/
            if z > &hist_max then z = &hist_max;

            /* Output simulated loss */
            gross_loss_sim = z;
            output;
        end;
    end;

    drop i u z;
run;
