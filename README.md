/* Should be rows_after = events_after if each event collapses to 1 row */
proc sql;
  create table compare_counts as
  select 
    (select events_before from raw_counts)  as events_before,
    (select rows_before   from raw_counts)  as rows_before,
    (select events_after  from after_counts) as events_after,
    (select rows_after    from after_counts) as rows_after
  from dictionary.tables
  where libname='WORK' and memname='AFTER_COUNTS';
quit;

title "Comparison of event and row counts before and after aggregation";
proc print data=compare_counts noobs; run;
