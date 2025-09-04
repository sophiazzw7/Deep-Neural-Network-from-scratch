/* --- Build unique (obligor, quarter) pairs --- */
proc sort data=mat_or_all out=_maa_;
  by c_obgobl OGM_Qtr;
run;

data _quarters_;
  set _maa_;
  by c_obgobl OGM_Qtr;
  if first.OGM_Qtr;              /* keep one row per (obligor, quarter) */
  keep c_obgobl OGM_Qtr;
run;

/* --- Count distinct quarters per obligor --- */
proc summary data=_quarters_ nway;
  class c_obgobl;
  output out=_rep_qtrs_(drop=_type_ _freq_) n=Qtr_Count;
run;

/* --- Build comma-separated quarter list per obligor --- */
proc sort data=_quarters_;
  by c_obgobl OGM_Qtr;
run;

data _rep_qtr_list_;
  length Quarter_List $60;
  retain Quarter_List;
  set _quarters_;
  by c_obgobl OGM_Qtr;
  if first.c_obgobl then Quarter_List='';
  Quarter_List = catx(',', Quarter_List, OGM_Qtr);
  if last.c_obgobl then output;
  keep c_obgobl Quarter_List;
run;

/* --- Total material-override events per obligor --- */
proc sql;
  create table _mat_counts_ as
  select c_obgobl, count(*) as Mat_OR_Count
  from mat_or_all
  group by c_obgobl
  ;
quit;

/* --- Final top_repeaters table (>=2 quarters) --- */
proc sql;
  create table top_repeaters as
  select r.c_obgobl,
         q.Qtr_Count,
         m.Mat_OR_Count,
         l.Quarter_List
  from repeated r
  left join _rep_qtrs_     q on r.c_obgobl=q.c_obgobl
  left join _mat_counts_   m on r.c_obgobl=m.c_obgobl
  left join _rep_qtr_list_ l on r.c_obgobl=l.c_obgobl
  order by q.Qtr_Count desc, m.Mat_OR_Count desc, r.c_obgobl
  ;
quit;

/* (Optional) peek */
title "Top Repeat Obligors (>=2 quarters)";
proc print data=top_repeaters(obs=50) noobs; run;
title;
