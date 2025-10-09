/* 1) See what unique_id really looks like */
proc contents data=cubes.bdfs_am_final_summary_202412; run;
proc sql outobs=20;
  select unique_id, company_id, start_date, lob_indicator, sample_tp
  from cubes.bdfs_am_final_summary_202412
  order by company_id, start_date;
quit;

/* 2) Count how many start_dates per unique_id (if >1 → unique_id isn’t per-month) */
proc sql;
  create table _uid_span as
  select unique_id,
         count(distinct start_date) as n_start_dates
  from cubes.bdfs_am_final_summary_202412
  group by unique_id
  having calculated n_start_dates > 1;
quit;

title "unique_id that covers multiple start_dates (not per-month)";
proc print data=_uid_span(obs=20); run;

/* 3) The only duplicate test that matters for the cube: business key */
proc sort data=cubes.bdfs_am_final_summary_202412
          out=_cube_dedup
          dupout=_cube_dups
          nodupkey;
  by company_id start_date lob_indicator sample_tp;  /* ETL’s key */
run;

title "TRUE duplicates on (company_id, start_date, lob_indicator, sample_tp)";
proc sql; select count(*) as true_dup_rows from _cube_dups; quit;

proc print data=_cube_dups(obs=20);
  var company_id start_date lob_indicator sample_tp score_value weight BAD_AT_12MO;
  format start_date date9.;
run;
