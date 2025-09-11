/* 1) How often is an obligor linked to multiple obligations in a quarter? */
proc sql;
  create table _multi_obgobl as
  select quarter, c_obg, count(distinct c_obgobl) as n_obligations
  from ABL_flags
  where not missing(c_obg)
  group by quarter, c_obg
  having calculated n_obligations > 1
  order by quarter, n_obligations desc;
quit;

/* 2) Compare repeat rates computed with the two different keys */
proc sort data=ABL_flags out=_k1; by c_obg qtr_idx; run;
proc sort data=ABL_flags out=_k2; by c_obgobl qtr_idx; run;

/* obligor-level (correct) */
data _k1; set _k1; by c_obg qtr_idx;
  retain prior_mat1 0;
  if first.c_obg then prior_mat1=0;
  repeat_mat_obg = (material_1notch=1 and prior_mat1>0);
  if material_1notch=1 then prior_mat1+1;
run;

/* obligation-level (likely error) */
data _k2; set _k2; by c_obgobl qtr_idx;
  retain prior_mat2 0;
  if first.c_obgobl then prior_mat2=0;
  repeat_mat_obgobl = (material_1notch=1 and prior_mat2>0);
  if material_1notch=1 then prior_mat2+1;
run;

/* 3) Quarter comparison */
proc sql;
  create table _cmp as
  select a.quarter,
         sum(a.material_1notch) as mat_total,
         sum(a.repeat_mat_obg)  as repeats_by_obg,
         sum(b.repeat_mat_obgobl) as repeats_by_obgobl
  from _k1 a
  join _k2 b
    on a.quarter=b.quarter and a.c_obgobl=b.c_obgobl and a.qtr_idx=b.qtr_idx
  group by a.quarter
  order by a.quarter;
quit;
