/* ---------- Global summary (macro-var + DATA step: safest) ------ */
proc sql noprint;
  /* totals from the material-override events table */
  select count(*), count(distinct c_obgobl)
    into :tot_mat, :distinct_obl
  from mat_or_all;

  /* count of repeat obligors (>=2 quarters) */
  select count(*) into :repeated_obl
  from repeated;
quit;

data rep_summary;
  length Total_Material_Overrides Distinct_Obligors_with_MatOR
         Repeated_Obligors_GE2Qtrs 8;
  Total_Material_Overrides       = &tot_mat.;
  Distinct_Obligors_with_MatOR   = &distinct_obl.;
  Repeated_Obligors_GE2Qtrs      = &repeated_obl.;
  if Total_Material_Overrides>0 then
    Share_of_All_MatOR = Repeated_Obligors_GE2Qtrs / Total_Material_Overrides;
  else Share_of_All_MatOR = .;
  format Share_of_All_MatOR percent8.2;
run;
