proc sql;
  select 'cube_n' as metric, count(*) as value from _cube_current
  union all
  select 'flat_n' as metric, count(*) as value from _flat_expected
  union all
  select 'matched_keys' as metric, count(*) as value
  from (
        select company_id, start_date
        from _cube_current
        intersect
        select company_id, start_date
        from _flat_expected
       );
quit;
