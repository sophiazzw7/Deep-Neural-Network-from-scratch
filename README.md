%let B=500;

data boot_parms;
length rep 8 par $20 estimate 8;
run;

%macro boot_et2;
%do rep=1 %to &B;

data severity_simulation_et2;
call streaminit(1234567890+&rep);
do i = 1 to &num_severity_simulation_needed;
    component_prob = ranuni(37534+&rep);
    if component_prob <= &mixing_probability_model1 then do;
        y=1000;
        do while (y<10000);
            y = rand("Exponential", &sigma_for_rand_exponential);
        end;
    end;
    else do;
        y=1000;
        do while (y<10000);
            y = rand("LOGNORMAL", &theta_for_rand_lognormal, &lambda_for_rand_lognormal);
        end;
    end;
    output;
end;
run;

ods exclude all;
ods output ParameterEstimates=tmp_parm;
proc fmm data=severity_simulation_et2 tech=newrap maxit=5000 gconv=0;
    model y = / dist=TruncExpo(10000,.);
    model y = / dist=LogNormal;
    probmodel;
run;
ods select all;

data tmp_parm2;
set tmp_parm;
length par $20;
if effect='Intercept' and component=1 then par='TE_intercept';
else if effect='Intercept' and component=2 then par='LN_intercept';
else if effect='Scale'     and component=2 then par='LN_scale';
if par='';
rep=&rep;
keep rep par estimate;
run;

proc append base=boot_parms data=tmp_parm2 force;
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
