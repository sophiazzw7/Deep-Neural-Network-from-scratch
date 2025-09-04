/* ---------- Global summary (robust, no CALCULATED in SELECT) --- */
proc sql;
  create table rep_summary as
  select
    tot                                 as Total_Material_Overrides,
    dist                                as Distinct_Obligors_with_MatOR,
    rpt                                 as Repeated_Obligors_GE2Qtrs,
    case when tot=0 then .
         else rpt / tot end             as Share_of_All_MatOR format=percent8.2
  from (
    select
      (select count(*)                         from mat_or_all)                as tot,
      (select count(distinct c_obgobl)         from mat_or_all)                as dist,
      (select count(*)                         from repeated)                  as rpt
  );
quit;
