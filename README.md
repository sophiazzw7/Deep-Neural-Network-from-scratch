%let rpt_prd_date_id=20241231;
%let eop12=%sysfunc(intnx(month,%sysfunc(inputn(&rpt_prd_date_id,yymmdd8.)),-12,end),date9.);
/* eop12 should be 31DEC2023 */
%put &=eop12;
2) Confirm the cube’s year mix (what you already saw)

sas
Copy code
proc sql;
  select sample_tp, vintage_yr, count(*) as n
  from cubes.bdfs_am_final_summary_202412
  group by sample_tp, vintage_yr
  order by sample_tp, vintage_yr;
quit;
3) Tie the FINAL cube back to the pieces you have

CURRENT slice should be a subset of the score feed with the same report cycle and start_date cut:

sas
Copy code
proc sql;
  /* current rows in the final cube */
  create table _cube_current as
  select *
  from cubes.bdfs_am_final_summary_202412
  where sample_tp='CURRENT';

  /* matching rows from the score feed under the same slicer */
  create table _flat_expected as
  select *
  from model.f_app_score_flat
  where rpt_prd_date_id=&rpt_prd_date_id
    and start_date <= "&eop12"d;

  /* overlap check on keys the ETL uses */
  select 
    (select count(*) from _cube_current) as cube_n,
    (select count(*) from _flat_expected) as flat_n,
    (select count(*) from 
        (select company_id, start_date from _cube_current
         intersect
         select company_id, start_date from _flat_expected)
    ) as matched_keys;
quit;
INTIME rows should match the INTIME sample by unique_id (or company_id/start_date if that’s how INTIME was built):

sas
Copy code
proc sql;
  select count(*) as intime_in_cube
  from cubes.bdfs_am_final_summary_202412
  where sample_tp='INTIME';

  /* If your INTIME table has unique_id: */
  select count(*) as intime_matched
  from cubes.bdfs_am_final_summary_202412 a
  inner join work.intime_sample_30jun2015 b
    on a.unique_id=b.unique_id
  where a.sample_tp='INTIME';
quit;
4) Duplicates using the ETL keys (how they join)

sas
Copy code
proc sort data=cubes.bdfs_am_final_summary_202412 out=_dedup dupout=_dups nodupkey;
  by company_id start_date lob_indicator sample_tp;
run;
proc sql; select count(*) as dups_on_etl_keys from _dups; quit;
If dups_on_etl_keys is ~0 (or negligible), duplication is just structural (multiple horizons), not an error.

How to slice only the 202412 OGM population (if that’s all you need)
sas
Copy code
data ogm_202412;
  set cubes.bdfs_am_final_summary_202412;
  where start_date <= "&eop12"d or sample_tp='INTIME';
run;
