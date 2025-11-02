/*==============================================================*
 * D) Duplicate checks on the event key ('Internal Events'n)
 *==============================================================*/

proc sort data=cleaned_data out=work._sorted_ noduprecs;
  by 'Internal Events'n;
run;

/* Capture duplicates into a separate table but do NOT print */
proc sort data=cleaned_data out=work._nodup_ nodupkey dupout=work._dups_;
  by 'Internal Events'n;
run;

/* Simple summary only */
title "Duplicate Check Summary (Internal Events)";
proc sql;
  select count(*) as total_rows format=comma18.,
         (select count(*) from work._dups_) as duplicate_rows format=comma18.,
         calculated duplicate_rows / calculated total_rows as pct_duplicates format=percent8.2
  from cleaned_data;
quit;
title;
