%let B = 1000;
%let num_severity_simulation_needed = 10000;

data boot_parms;
length rep 8 par $32.;
run;

%macro boot_et2;
%do rep = 1 %to &B;

data severity_simulation_et2;
call streaminit(1234567890+&rep);
do i = 1 to &num_severity_simulation_needed;
    component_prob = ranuni(37534+&rep);
    if component_prob <= &mixing_probability_model1 then do;
        y = 1000;
        do while (y < 10000);
            y = rand("Exponential", &sigma_for_rand_exponential);
        end;
    end;
    else do;
        y = 1000;
        do while (y < 10000);
            y = rand("LOGNORMAL", &theta_for_rand_lognormal, &lambda_for_rand_lognormal);
        end;
    end;
    output;
end;
run;

ods exclude all;
ods output ParameterEstimates=parms_one;
proc fmm data=severity_simulation_et2;
    model y = / dist=exp;
    model y = / dist=logn;
    probmodel;
run;
ods select all;

data parms_one2;
set parms_one;
length par $32.;
if effect='Intercept' and component=1 then par='exp_intercept';
else if effect='Intercept' and component=2 then par='ln_mu';
else if effect='Scale'     and component=2 then par='ln_scale';
else if parameter='Probability 1'          then par='mix_p1';
if par ne '';
rep = &rep;
keep rep par estimate;
run;

proc append base=boot_parms data=parms_one2 force;
run;

%end;
%mend;

%boot_et2;

proc sort data=boot_parms;
by par;
run;

proc univariate data=boot_parms noprint;
by par;
var estimate;
output out=boot_ci pctlpts=2.5 97.5 pctlpre=p_;
run;
