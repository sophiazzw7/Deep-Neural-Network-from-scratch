%let B = 200;      /* number of bootstrap samples */
%let L_trunc = 10000;

proc datasets lib=work nolist;
    delete boot_param_: boot_all;
quit;

%macro boot_et2;

%do b = 1 %to &B.;

    /* 1) Resample with replacement using only a DATA step */
    data boot_sample;
        if 0 then set severity_et2_10k nobs=nobs;
        call streaminit(12345 + &b);

        do i = 1 to nobs;
            pt = ceil( rand("Uniform") * nobs );
            set severity_et2_10k point=pt;
            output;
        end;
        stop;
    run;

    /* 2) Fit exactly the same model as developer */
    proc fmm data=boot_sample;
        model gross_loss = / dist=TruncExpo(&L_trunc);
        model gross_loss = / dist=LogNormal;
        probmodel;
        ods output ParameterEstimates=boot_param_&b;
    run;

    /* 3) Add replicate id and stack */
    data boot_param_&b;
        set boot_param_&b;
        Replicate = &b;
    run;

    proc append base=boot_all data=boot_param_&b force; run;

%end;

%mend;

%boot_et2;
