/* TRUE duplicate check in the cube using GROUP BY */
proc sql;
  create table cube_true_dupgroups as
  select company_id, start_date, lob_indicator, sample_tp,
         count(*) as n_rows
  from cubes.bdfs_am_final_summary_202412
  group by company_id, start_date, lob_indicator, sample_tp
  having calculated n_rows > 1;
quit;

/* How many duplicate key-groups? */
proc sql; select count(*) as n_dup_key_groups from cube_true_dupgroups; quit;

/* Show sample duplicate groups + their rows */
proc sql;
  /* list first few groups */
  select * from cube_true_dupgroups(obs=10);
quit;

proc sql;
  /* pull the actual duplicate rows for those groups */
  create table cube_true_dups as
  select a.*
  from cubes.bdfs_am_final_summary_202412 a
  inner join cube_true_dupgroups g
    on  a.company_id   = g.company_id
    and a.start_date   = g.start_date
    and a.lob_indicator= g.lob_indicator
    and a.sample_tp    = g.sample_tp
  order by a.company_id, a.start_date, a.lob_indicator, a.sample_tp;
quit;

title "Actual duplicate rows on (company_id, start_date, lob_indicator, sample_tp)";
proc print data=cube_true_dups(obs=30);
  var company_id start_date lob_indicator sample_tp score_value weight BAD_AT_12MO;
  format start_date date9.;
run;
