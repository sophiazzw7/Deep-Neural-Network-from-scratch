/* Parameters */
%let rpt_prd_date_id = 20241231;
%let eop12 = %sysfunc(intnx(month,%sysfunc(inputn(&rpt_prd_date_id,yymmdd8.)),-12,end),date9.);  /* 31DEC2023 */

/* Cube slices (unchanged) */
data _cube_current _cube_intime;
  set cubes.bdfs_am_final_summary_202412;
  if sample_tp='CURRENT' and start_date <= "&eop12"d then output _cube_current;
  else if sample_tp='INTIME' then output _cube_intime;
run;

/* f_app_score_flat: build start_date = EOM(score_date) with fallback to eff_from_dt */
data _flat_expected;
  set model.f_app_score_flat;
  length start_date 8; format start_date date9.;
  /* derive EOM from score_date; if missing, fallback to eff_from_dt */
  start_date = ifn( not missing(score_date)
                   , intnx('month', score_date, 0, 'end')
                   , eff_from_dt );
  where sample_tp='CURRENT' and start_date <= "&eop12"d;
  keep company_id start_date score_date eff_from_dt score_value weight sample_tp;
run;

/* Keys & overlap (character company_id) */
data _keys_cube(keep=key) _keys_flat(keep=key);
  length key $64;
  set _cube_current(in=a) _flat_expected(in=b);
  if a then key = cats(strip(company_id),'|',put(start_date,yymmddn8.));
  else if b then key = cats(strip(company_id),'|',put(start_date,yymmddn8.));
  if a then output _keys_cube;
  if b then output _keys_flat;
run;

proc sql;
  title "Overlap (company_id, start_date): cube CURRENT vs f_app EXPECTED";
  select 'cube_n' as metric length=16, count(*) as value from _keys_cube
  union all select 'flat_n', count(*) from _keys_flat
  union all select 'matched_keys', count(*) 
  from (select key from _keys_cube intersect select key from _keys_flat);
quit;

/* Duplicates */
proc sort data=_cube_current out=_cc_dedup dupout=_cc_dups nodupkey;
  by company_id start_date lob_indicator sample_tp;
run;
proc sql; title "Cube CURRENT dups"; select count(*) as dup_rows from _cc_dups; quit;

proc sort data=_flat_expected out=_fe_dedup dupout=_fe_dups nodupkey;
  by company_id start_date;
run;
proc sql; title "f_app EXPECTED dups"; select count(*) as dup_rows from _fe_dups; quit;
