/* 在宏内部 %OR 的最后追加以下内容 */

/* ================== Material Override Summary ================== */
proc sql;
  create table Material_Summary_&Seg as
  select
    "&OGM_Qtr" as Quarter length=6,
    "&Seg"     as Segment length=3,
    count(*) as Total_Obs,
    sum(override_ind) as Overrides,
    sum(case when abs(OVRD_SEVERITY) > 1 then override_ind else 0 end) as Material_Overrides,
    calculated Overrides / calculated Total_Obs format=percent8.2 as Override_Pct,
    calculated Material_Overrides / calculated Total_Obs format=percent8.2 as Material_Pct
  from Override_&OGM_Qtr._dedup_&Seg
  ;
quit;

title "Material Override Rate (&OGM_Qtr - &Seg)";
proc print data=Material_Summary_&Seg noobs; run;
title;

/* ================== Override Reason Distribution ================ */
proc sql;
  create table Reason_Distribution_&Seg as
  select
    "&OGM_Qtr" as Quarter length=6,
    "&Seg"     as Segment length=3,
    Override_Reason,
    count(*) as cnt,
    sum(case when abs(OVRD_SEVERITY)>1 then 1 else 0 end) as material_cnt
  from Override_&OGM_Qtr._dedup_&Seg
  group by Override_Reason
  order by calculated cnt desc
  ;
quit;

title "Override Reasons (&OGM_Qtr - &Seg)";
proc print data=Reason_Distribution_&Seg noobs; run;
title;

/* ================== Repeat Obligor Across Quarters =============== */
/* 注意：要在所有季度都跑完 %OR 之后再执行以下代码一次即可 */
%mend OR;

/* ---------- 在跑完所有季度后，加下面的跨季度 repeat 检查 ---------- */
proc sql;
  create table Repeat_Obligor as
  select 
    c_obgobl,
    Segment,
    count(distinct OGM_Qtr) as n_quarters_overridden,
    min(OGM_Qtr) as first_qtr,
    max(OGM_Qtr) as last_qtr,
    sum(case when abs(OVRD_SEVERITY)>1 then 1 else 0 end) as n_material_overrides
  from (
        select * from Override_2024Q2_dedup_all
        union corr
        select * from Override_2024Q3_dedup_all
        union corr
        select * from Override_2024Q4_dedup_all
        union corr
        select * from Override_2025Q1_dedup_all
       )
  where override_ind=1
  group by c_obgobl, Segment
  having calculated n_quarters_overridden >= 2
  order by n_quarters_overridden desc, c_obgobl;
quit;

title "Repeat Obligors with Material Overrides Across Quarters";
proc print data=Repeat_Obligor(obs=50) noobs; run;
title;
