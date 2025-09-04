/*****************************************************/
/* Top Repeaters (robust, no PROC SUMMARY/PRX)       */
/* Inputs required:                                   */
/*   - mat_or_all  : rows of material overrides       */
/*                   (vars: c_obgobl, OGM_Qtr, ...)   */
/*   - repeated    : obligors with Qtr_Count > 1      */
/*****************************************************/

/* 1) Unique (obligor, quarter) pairs */
proc sql;
  create table _quarters_ as
  select distinct c_obgobl, OGM_Qtr
  from mat_or_all
  where not missing(c_obgobl) and not missing(OGM_Qtr)
  ;
quit;

/* 2) Count distinct quarters per obligor */
proc sql;
  create table _rep_qtrs_ as
  select c_obgobl,
         count(*) as Qtr_Count
  from _quarters_
  group by c_obgobl
  ;
quit;

/* 3) Build comma-separated quarter list per obligor */
proc sort data=_quarters_;
  by c_obgobl OGM_Qtr;
run;

data _rep_qtr_list_;
  length Quarter_List $100;
  retain Quarter_List;
  set _quarters_;
  by c_obgobl OGM_Qtr;
  if first.c_obgobl then Quarter_List = '';
  Quarter_List = catx(',', Quarter_List, OGM_Qtr);
  if last.c_obgobl then output;
  keep c_obgobl Quarter_List;
run;

/* 4) Total material-override EVENTS per obligor */
proc sql;
  create table _mat_counts_ as
  select c_obgobl,
         count(*) as Mat_OR_Count
  from mat_or_all
  group by c_obgobl
  ;
quit;

/* 5) Final Top Repeaters table (>=2 quarters) */
proc sql;
  create table top_repeaters as
  select r.c_obgobl,
         q.Qtr_Count,
         m.Mat_OR_Count,
         l.Quarter_List
  from repeated r
  left join _rep_qtrs_     q on r.c_obgobl = q.c_obgobl
  left join _mat_counts_   m on r.c_obgobl = m.c_obgobl
  left join _rep_qtr_list_ l on r.c_obgobl = l.c_obgobl
  order by q.Qtr_Count desc, m.Mat_OR_Count desc, r.c_obgobl
  ;
quit;

/* 6) Optional peek */
title "Top Repeat Obligors (>=2 quarters)";
proc print data=top_repeaters(obs=50) noobs; run;
title;
