/* 1) Duplicate check on unique_id (cube) */
proc sort data=cubes.bdfs_am_final_summary_202412
          out=_cube_uid_dedup
          dupout=_cube_uid_dups
          nodupkey;
  by unique_id;
run;

title "Duplicates in cube on unique_id";
proc sql; select count(*) as dup_rows from _cube_uid_dups; quit;

proc print data=_cube_uid_dups (obs=20);
  var unique_id company_id start_date lob_indicator sample_tp score_value weight BAD_AT_12MO;
  format start_date date9.;
run;

/* 2) (Optional) enforce unique_id de-dup for a working copy */
data cube_nodups_uniqueid;
  set _cube_uid_dedup;  /* 1 row per unique_id */
run;
/* Flat expected (CURRENT, <= eop12) assumed already built as _flat_expected */
proc sort data=_flat_expected out=_flat_uid_dedup dupout=_flat_uid_dups nodupkey;
  by unique_id;
run;

title "Duplicates in f_app_score_flat slice on unique_id";
proc sql; select count(*) as dup_rows from _flat_uid_dups; quit;

proc print data=_flat_uid_dups (obs=20);
  var unique_id company_id start_date score_date eff_from_dt score_value weight;
  format start_date date9.;
run;
