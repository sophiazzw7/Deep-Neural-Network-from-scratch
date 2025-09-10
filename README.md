proc sql;
  create table direction_consistent as
  select quarter,
         sum(override_ind=1)                                            as overrides,
         sum(material_1notch=1 and OVRD_SEVERITY>0)                     as mat_downgrades,
         sum(material_1notch=1 and OVRD_SEVERITY<0)                     as mat_upgrades,
         sum(material_1notch=1)                                         as mat_total
  from _abl_enriched
  group by quarter
  order by quarter;
quit;

title "Direction (consistent with material totals)";
proc print data=direction_consistent noobs;
  var quarter overrides mat_downgrades mat_upgrades mat_total;
run;


proc sql;
  create table _flags_join2 as
  select f.quarter, f.c_obgobl, f.material_1notch, f.repeat_material, f.qtr_idx,
         a.c_obg, a.tfc_face_amt, a.TFC_Curr_Bal,
         a.legacy_bank_new, a.STI_lob_nm, a.STI_sub_lob_nm
  from _abl_flags f
  left join _abl_attrs a
    on f.quarter=a.quarter and f.c_obgobl=a.c_obgobl;
quit;

/* obligor-level view across quarters */
proc sql;
  create table obg_q as
  select c_obg, quarter,
         max(material_1notch) as mat_obg,
         max(repeat_material) as rpt_obg,
         sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end) as bal_material,
         sum(case when material_1notch=1 then tfc_face_amt else 0 end) as face_material
  from _flags_join2
  group by c_obg, quarter;
quit;

proc sql;
  create table repeat_obligor_rank as
  select c_obg,
         sum(mat_obg) as mat_qtrs,
         sum(rpt_obg) as rpt_qtrs,
         sum(face_material) as face_material_total,
         sum(bal_material)  as bal_material_total
  from obg_q
  group by c_obg
  having rpt_qtrs>=1
  order by bal_material_total desc;
quit;

proc sort data=obg_q; by c_obg quarter; run;
data repeat_obligor_quarters;
  set obg_q;
  by c_obg quarter;
  length quarters $200;
  retain quarters;
  if first.c_obg then quarters='';
  if mat_obg=1 then quarters=catx(',',quarters,quarter);
  if last.c_obg then output;
  keep c_obg quarters;
run;

proc sql;
  create table top_repeating_obligors as
  select a.*, b.quarters
  from repeat_obligor_rank a
  left join repeat_obligor_quarters b
    on a.c_obg=b.c_obg
  order by bal_material_total desc;
quit;

/* repeat share contribution by heritage and LOB */
proc sql;
  create table _rpt_totals_qtr as
  select quarter, sum(repeat_material) as rpt_total_qtr
  from _flags_join2
  group by quarter;
quit;

proc sql;
  create table repeat_share_heritage_qtr as
  select a.quarter, coalesce(a.legacy_bank_new,'(missing)') as heritage,
         sum(a.repeat_material) as rpt_total,
         sum(a.repeat_material)/b.rpt_total_qtr as rpt_share format=percent8.2
  from _flags_join2 a
  join _rpt_totals_qtr b
    on a.quarter=b.quarter
  group by a.quarter, calculated heritage, b.rpt_total_qtr
  order by a.quarter, rpt_share desc;
quit;

proc sql;
  create table repeat_share_lob_qtr as
  select a.quarter, coalesce(a.STI_lob_nm,'(missing)') as lob,
         sum(a.repeat_material) as rpt_total,
         sum(a.repeat_material)/b.rpt_total_qtr as rpt_share format=percent8.2
  from _flags_join2 a
  join _rpt_totals_qtr b
    on a.quarter=b.quarter
  group by a.quarter, calculated lob, b.rpt_total_qtr
  order by a.quarter, rpt_share desc;
quit;

proc sql;
  create table repeat_share_heritage_all as
  select coalesce(legacy_bank_new,'(missing)') as heritage,
         sum(repeat_material) as rpt_total,
         sum(repeat_material)/(select sum(repeat_material) from _flags_join2) as rpt_share format=percent8.2
  from _flags_join2
  group by calculated heritage
  order by rpt_total desc;
quit;

proc sql;
  create table repeat_share_lob_all as
  select coalesce(STI_lob_nm,'(missing)') as lob,
         sum(repeat_material) as rpt_total,
         sum(repeat_material)/(select sum(repeat_material) from _flags_join2) as rpt_share format=percent8.2
  from _flags_join2
  group by calculated lob
  order by rpt_total desc;
quit;

/* prints */
title "Top Repeating Obligors by Balance";
proc print data=top_repeating_obligors(obs=20) noobs label;
  var c_obg rpt_qtrs mat_qtrs bal_material_total face_material_total quarters;
  label rpt_qtrs='repeat_qtrs' mat_qtrs='material_qtrs' 
        bal_material_total='bal_material' face_material_total='face_material';
run;

title "Repeat Share by Heritage (quarter)";
proc print data=repeat_share_heritage_qtr noobs;
  var quarter heritage rpt_total rpt_share;
run;

title "Repeat Share by LOB (quarter)";
proc print data=repeat_share_lob_qtr noobs;
  var quarter lob rpt_total rpt_share;
run;

title "Repeat Share by Heritage (window total)";
proc print data=repeat_share_heritage_all noobs;
  var heritage rpt_total rpt_share;
run;

title "Repeat Share by LOB (window total)";
proc print data=repeat_share_lob_all noobs;
  var lob rpt_total rpt_share;
run;

title;
