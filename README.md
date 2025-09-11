Thanks—your error means `_abl_flags` doesn’t have `c_obg`. The safest fix is to **pull `c_obg` from `_abl_attrs` via `c_obgobl`** and then proceed. Paste the blocks below **right after** you’ve created `_abl_flags` and `_abl_attrs`.

---

### 0) Create a clean joined table (also fixes missing `c_obg`)

```sas
/* Join flags ↔ attributes; source c_obg from attrs when missing */
proc sql;
  create table _flags_join2 as
  select
      f.quarter,
      coalesce(f.c_obg, a.c_obg)                 as c_obg,        /* obligor key (for repeats) */
      f.c_obgobl,                                                /* obligor+obligation key */
      f.qtr_idx,
      f.material_1notch,
      f.repeat_material,
      f.OVRD_SEVERITY,
      a.tfc_face_amt,
      a.TFC_Curr_Bal,
      a.legacy_bank_new,
      a.STI_lob_nm,
      a.STI_sub_lob_nm,
      a.Override_Reason
  from _abl_flags f
  left join _abl_attrs a
    on  f.quarter  = a.quarter
    and f.c_obgobl = a.c_obgobl
  ;
quit;
```

---

### 1) Direction (material-only)

```sas
proc sql;
  create table direction_consistent as
  select
      quarter,
      sum(material_1notch=1)                          as mat_total,
      sum(material_1notch=1 and OVRD_SEVERITY>0)      as mat_downgrades,
      sum(material_1notch=1 and OVRD_SEVERITY<0)      as mat_upgrades
  from _flags_join2
  group by quarter
  order by quarter;
quit;

title "Direction (consistent with material totals)";
proc print data=direction_consistent noobs;
  var quarter mat_downgrades mat_upgrades mat_total;
run; title;
```

---

### 2) Exposure share (obligation level via `c_obgobl`)

```sas
proc sql;
  create table abl_exposure_share as
  select
      quarter,
      sum(tfc_face_amt)                                              as face_total,
      sum(case when material_1notch=1 then tfc_face_amt  else 0 end) as face_material,
      calculated face_material / max(calculated face_total,1)        as face_mat_share format=percent8.2,
      sum(TFC_Curr_Bal)                                              as bal_total,
      sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end)  as bal_material,
      calculated bal_material / max(calculated bal_total,1)          as bal_mat_share  format=percent8.2
  from _flags_join2
  group by quarter
  order by quarter;
quit;

title "Exposure Share of Material";
proc print data=abl_exposure_share noobs;
  var quarter face_total face_material face_mat_share bal_total bal_material bal_mat_share;
run; title;
```

---

### 3) Reasons (top-10 by quarter) + **optional repeats-only**

```sas
/* All material reasons (per quarter) */
proc sql;
  create table abl_by_reason as
  select
      quarter,
      coalesce(Override_Reason,'(missing)')           as reason length=200,
      count(*)                                        as obs,
      sum(material_1notch)                            as mat_cnt,
      calculated mat_cnt / max(calculated obs,1)      as mat_rate format=percent8.2
  from _flags_join2
  group by quarter, reason
  order by quarter, mat_cnt desc;
quit;

title "Material Override Reasons (top 10 by material count)";
proc sort data=abl_by_reason out=_t3; by quarter descending mat_cnt; run;
proc print data=_t3(obs=10) noobs; var quarter reason obs mat_cnt mat_rate; run; title;

/* OPTIONAL: reasons among repeats only */
proc sql;
  create table abl_by_reason_repeats as
  select
      quarter,
      coalesce(Override_Reason,'(missing)')           as reason length=200,
      count(*)                                        as obs,
      sum(material_1notch and repeat_material)        as rpt_mat_cnt,
      calculated rpt_mat_cnt / max(calculated obs,1)  as rpt_mat_rate format=percent8.2
  from _flags_join2
  group by quarter, reason
  order by quarter, rpt_mat_cnt desc;
quit;

title "Material Reasons among Repeat Overrides (top 10)";
proc sort data=abl_by_reason_repeats out=_t3r; by quarter descending rpt_mat_cnt; run;
proc print data=_t3r(obs=10) noobs; var quarter reason obs rpt_mat_cnt rpt_mat_rate; run; title;
```

---

### 4) Repeat overrides by quarter

```sas
proc sql;
  create table abl_material_repeat_by_qtr as
  select
      quarter,
      count(*)                                                as total_observations,
      sum(material_1notch=1)                                  as mat_total,
      sum(material_1notch=1 and repeat_material=1)            as mat_repeat,
      calculated mat_repeat / max(calculated mat_total,1)     as mat_repeat_pct format=percent8.2,
      sum(material_1notch=1 and repeat_material=0)            as mat_first_time,
      calculated mat_first_time / max(calculated mat_total,1) as mat_first_time_pct format=percent8.2
  from _flags_join2
  group by quarter
  order by quarter;
quit;

title "Material Overrides Repeat by Quarter";
proc print data=abl_material_repeat_by_qtr noobs label;
  var quarter total_observations mat_total mat_repeat mat_repeat_pct mat_first_time mat_first_time_pct;
run; title;
```

---

### 5) Top repeating obligors by balance (uses `c_obg` now available)

```sas
/* Obligor-quarter aggregates */
proc sql;
  create table _obg_q as
  select
      c_obg,
      quarter,
      max(material_1notch)                           as mat_obg,
      max(repeat_material)                           as rpt_obg,
      sum(case when material_1notch=1 then TFC_Curr_Bal else 0 end) as bal_material,
      sum(case when material_1notch=1 then tfc_face_amt  else 0 end) as face_material
  from _flags_join2
  group by c_obg, quarter;
quit;

/* Roll up across window */
proc sql;
  create table repeat_obligor_rank as
  select
      c_obg,
      sum(mat_obg)           as mat_qtrs,
      sum(rpt_obg)           as rpt_qtrs,
      sum(face_material)     as face_material_total,
      sum(bal_material)      as bal_material_total
  from _obg_q
  group by c_obg
  having rpt_qtrs >= 1
  order by bal_material_total desc;
quit;

/* Quarter list for context */
proc sort data=_obg_q; by c_obg quarter; run;
data repeat_obligor_quarters;
  set _obg_q;
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
    on a.c_obg = b.c_obg
  order by bal_material_total desc;
quit;

title "Top Repeating Obligors by Balance";
proc print data=top_repeating_obligors(obs=20) noobs label;
  var c_obg rpt_qtrs mat_qtrs bal_material_total face_material_total quarters;
  label rpt_qtrs='repeat_qtrs' mat_qtrs='material_qtrs'
        bal_material_total='bal_material' face_material_total='face_material';
run; title;
```

---

**Why this fixes your error:** we **coalesce** `c_obg` from `_abl_attrs` when it’s missing in `_abl_flags`, so all downstream steps that group by `c_obg` will run without altering your earlier pipeline.
