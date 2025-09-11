/* -- A1. Make a clean flags table with obligor & obligor+obligation keys -- */
proc sql;
  create table _flags_fix as
  select
      /* obligor id (prefer c_obg; else obligor_key) */
      coalesce(f.c_obg, f.obligor_key)              as c_obg length=200,
      f.quarter,
      f.qtr_idx,
      f.material_1notch,
      f.repeat_material,
      f.OVRD_SEVERITY,
      /* obligor+obligation; prefer existing c_obgobl; if absent, synthesize from obligor + facility */
      coalesce(f.c_obgobl,
               cats(coalesce(f.c_obg, f.obligor_key),'|',coalesce(f.LASFACILITYNBR,''))) as c_obgobl length=300
  from ABL_flags as f
  ;
quit;

/* -- A2. Make a clean attributes table with the same keys -- */
proc sql;
  create table _attrs_fix as
  select
      a.quarter,
      /* carry both keys for safety */
      coalesce(a.c_obg, a.obligor_key)              as c_obg length=200,
      coalesce(a.c_obgobl,
               cats(coalesce(a.c_obg, a.obligor_key),'|',coalesce(a.LASFACILITYNBR,''))) as c_obgobl length=300,
      a.tfc_face_amt,
      a.TFC_Curr_Bal,
      a.legacy_bank_new,
      a.STI_lob_nm,
      a.STI_sub_lob_nm,
      a.Override_Reason
  from _abl_attrs as a
  ;
quit;

/* -- A3. Unified table for all downstream calcs -- */
proc sql;
  create table _flags_join2 as
  select
      f.quarter,
      f.c_obg,
      f.c_obgobl,
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
  from _flags_fix f
  left join _attrs_fix a
    on  f.quarter  = a.quarter
    and f.c_obgobl = a.c_obgobl
  ;
quit;

/* (Optional) quick sanity checks if integration still fails:
proc contents data=_flags_fix; run;
proc contents data=_attrs_fix; run;
proc print data=_flags_join2(obs=5); run;
*/
