options compress=yes reuse=yes;

/* 1) Point each quarter lib to its folder */
libname q2 "/sasdata/.../2024Q2_v2.1";
libname q3 "/sasdata/.../2024Q3_v2.1";
libname q4 "/sasdata/.../2024Q4_v2.1";
libname q1 "/sasdata/.../2025Q1_v2.2";

/* 2) Tell SAS which dataset name lives in which lib and what quarter label to use */
data _tables;
  length lib $8 memname $32 quarter $7;
  infile datalines truncover;
  input lib $ memname $ quarter $;
datalines;
q2 all_table_2406_1 2024Q2
q3 all_table_2409_1 2024Q3
q4 all_table_2412_1 2024Q4
q1 all_table_2503_1 2025Q1
;
run;

/* 3) Run the QC pack for each table (developer keys) */
data _null_;
  set _tables;
  length ds $64;
  ds = cats(lib,'.',memname);

  call execute("proc sql; create table qc_rowcount_"||quarter||" as
                 select '"||quarter||"' as quarter length=7,
                        count(*) as rows,
                        count(distinct TFC_Account_Nbr_New) as distinct_acct_new,
                        count(distinct cats(put(c_obg,z10.),put(c_obl,z5.))) as distinct_c_obgobl
                 from "||ds||"; quit;");

  call execute("proc sql; create table qc_dups_acct_"||quarter||" as
                 select TFC_Account_Nbr_New, count(*) as n
                 from "||ds||"
                 group by TFC_Account_Nbr_New
                 having n>1; quit;");

  call execute("proc sql; create table qc_dups_obgobl_"||quarter||" as
                 select cats(put(c_obg,z10.),put(c_obl,z5.)) as c_obgobl length=15, count(*) as n
                 from "||ds||"
                 group by calculated c_obgobl
                 having n>1; quit;");

  call execute("proc sql; create table qc_dups_pair_"||quarter||" as
                 select cats(put(c_obg,z10.),put(c_obl,z5.)) as c_obgobl length=15,
                        TFC_Account_Nbr_New,
                        count(*) as n
                 from "||ds||"
                 group by calculated c_obgobl, TFC_Account_Nbr_New
                 having n>1; quit;");

  call execute("proc sql; create table qc_missing_"||quarter||" as
                 select '"||quarter||"' as quarter length=7,
                        sum(missing(TFC_Account_Nbr_New)) as miss_acct_new,
                        sum(missing(c_obg)) as miss_c_obg,
                        sum(missing(c_obl)) as miss_c_obl,
                        sum(missing(TFC_Curr_Bal)) as miss_curr_bal,
                        sum(missing(tfc_face_amt)) as miss_face_amt,
                        sum(missing(STI_rating_model)) as miss_rating_model,
                        sum(missing(legacy_bank_new)) as miss_heritage,
                        sum(missing(business_group)) as miss_business_group,
                        sum(missing(STI_lob_nm)) as miss_lob,
                        sum(missing(STI_sub_lob_nm)) as miss_sub_lob
                 from "||ds||"; quit;");

  call execute("proc sql; create table qc_ranges_"||quarter||" as
                 select '"||quarter||"' as quarter length=7,
                        sum(TFC_Curr_Bal<0) as neg_curr_bal,
                        sum(tfc_face_amt<0) as neg_face_amt,
                        sum(STI_BND_EXPOS_AMT<0) as neg_bnd_expos_amt,
                        sum(final_lgd_perc<0 or final_lgd_perc>100) as bad_final_lgd_0_100,
                        sum(model_lgd_perc<0 or model_lgd_perc>100) as bad_model_lgd_0_100,
                        sum(over_lgd_perc<0 or over_lgd_perc>100) as bad_over_lgd_0_100,
                        sum(final_lgd_perc between 0 and 1)  as final_lgd_in_0_1,
                        sum(model_lgd_perc between 0 and 1)  as model_lgd_in_0_1,
                        sum(over_lgd_perc  between 0 and 1)  as over_lgd_in_0_1
                 from "||ds||"; quit;");

  call execute("proc means data="||ds||" n nmiss min p1 p5 p50 p95 p99 max;
                 var TFC_Curr_Bal tfc_face_amt STI_BND_EXPOS_AMT final_lgd_perc model_lgd_perc over_lgd_perc;
                 title 'QC Numeric Distributions "||quarter||"'; run; title;");

  call execute("proc sort data="||ds||" out=_topbal_"||quarter||"(keep=TFC_Account_Nbr_New TFC_Curr_Bal); by descending TFC_Curr_Bal; run;");
  call execute("proc print data=_topbal_"||quarter||"(obs=20) noobs; title 'Top 20 Current Balance "||quarter||"'; run; title;");

  call execute("proc sort data="||ds||" out=_topface_"||quarter||"(keep=TFC_Account_Nbr_New tfc_face_amt); by descending tfc_face_amt; run;");
  call execute("proc print data=_topface_"||quarter||"(obs=20) noobs; title 'Top 20 Face Amount "||quarter||"'; run; title;");

  call execute("proc freq data="||ds||" nlevels order=freq;
                 tables STI_rating_model legacy_bank_new business_group STI_lob_nm STI_sub_lob_nm / missing;
                 title 'QC Categorical "||quarter||"'; run; title;");

  call execute("data _util_"||quarter||"; set "||ds||";
                 length c_obgobl $15; c_obgobl=cats(put(c_obg,z10.),put(c_obl,z5.));
                 if tfc_face_amt>0 then util=TFC_Curr_Bal/tfc_face_amt; else util=.;
                 if util<0 then util=0; if util>2 then util=2; run;");
  call execute("proc means data=_util_"||quarter||" n nmiss min p5 p50 p95 max; var util; title 'Utilization Summary "||quarter||"'; run; title;");
run;

/* 4) Collate quick summaries across quarters */
data qc_rowcounts_all; set qc_rowcount_:; run;
data qc_missing_all;   set qc_missing_:;   run;
data qc_ranges_all;    set qc_ranges_:;    run;

title "QC Row Counts (all quarters)";
proc print data=qc_rowcounts_all noobs; run;

title "QC Missingness (all quarters)";
proc print data=qc_missing_all noobs; run;

title "QC Range and Scale Checks (all quarters)";
proc print data=qc_ranges_all noobs; run;

title;
