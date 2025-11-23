%macro boot_sev_et2(B=500);

    /* Container for all bootstrap parameter estimates */
    proc datasets lib=work nolist;
        delete boot_param_all;
    quit;

    %do b = 1 %to &B.;

        /* 1) Resample ET2 severities with replacement */
        proc surveyselect data=severity_et2_10k
            out=boot_sample_et2
            method=urs         /* unrestricted sampling with replacement */
            samprate=1         /* same size as original */
            outhits
            seed=&b;
        run;

        /* 2) Refit the mixture model on the bootstrap sample */
        /* IMPORTANT: make this PROC FMM block match your original model exactly */
        proc fmm data=boot_sample_et2;
            model gross_loss = / dist=exponential;
            model gross_loss = / dist=lognormal;
            probmodel;
            ods output ParameterEstimates=boot_param_&b;
        run;

        /* 3) Add replicate ID and stack into one dataset */
        data boot_param_&b;
            set boot_param_&b;
            Replicate = &b;
        run;

        proc append base=boot_param_all data=boot_param_&b force; run;

    %end;

%mend;

/* Run with, say, 500 or 1000 replicates */
%boot_sev_et2(B=500);

/* Keep only the parameters you care about (optional filter) */
/* For FMM, typical Parameter names: Intercept (Exp), Intercept (Lognormal),
   Scale (Lognormal), and mixing probs inside the ProbModel. Adjust as needed. */

proc sort data=boot_param_all;
    by Parameter;
run;

proc univariate data=boot_param_all noprint;
    by Parameter;
    var Estimate;
    output out=sev_et2_param_CI
        pctlpts = 2.5 97.5
        pctlpre = CI_;
run;

proc print data=sev_et2_param_CI;
    title "Bootstrap 95% CIs for ET2 Severity Mixture Parameters";
run;
