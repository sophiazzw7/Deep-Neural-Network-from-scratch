/* =========================== SETUP =========================== */
/* EDIT THIS ROOT ONLY (folders under it must exist as named)   */
%let ROOT=/sasdata/mrmg1/MOD1497;

/* Map each quarter to its folder */
libname q2  "&ROOT/2024Q2_v2.1";
libname q3  "&ROOT/2024Q3_v2.1";
libname q4  "&ROOT/2024Q4_v2.1";
libname q1  "&ROOT/2025Q1_v2.2";

/* ---------------- Helper: letter grade -> numeric rank ------- */
proc format;
  value $lgd_num
    'A'=1  'B'=2  'C'=3  'D'=4  'E'=5
    'F'=6  'G'=7  'H'=8  'I'=9  'J'=10
    'K'=11 'L'=12 'M'=13 'N'=14 'O'=15
    'P'=16 'Q'=17 'R'=18 'S'=19 'T'=20
  ;
run;

/* ================ 1) STACK QUARTERS + CLEAN ================== */
/* Uses the production OGM population by segment for each quarter */
data OV_DISC_RAW;
  length OGM_Qtr $6 segment $3 facility_key $60 source_path $200;
  set
    /* 2024Q2 */
    q2.join_mod1497_tot_2024q2_abl (in=a1) 
    q2.join_mod1497_tot_2024q2_sf  (in=a2)
    /* 2024Q3 */
    q3.join_mod1497_tot_2024q3_abl (in=b1)
    q3.join_mod1497_tot_2024q3_sf  (in=b2)
    /* 2024Q4 */
    q4.join_mod1497_tot_2024q4_abl (in=c1)
    q4.join_mod1497_tot_2024q4_sf  (in=c2)
    /* 2025Q1 */
    q1.join_mod1497_tot_2025q1_abl (in=d1)
    q1.join_mod1497_tot_2025q1_sf  (in=d2)
  ;

  /* Tag quarter and segment */
  if a1 or a2 then OGM_Qtr='2024Q2';
  else if b1 or b2 then OGM_Qtr='2024Q3';
  else if c1 or c2 then OGM_Qtr='2024Q4';
  else if d1 or d2 then OGM_Qtr='2025Q1';

  segment = ifc(a2 or b2 or c2 or d2,'SF','ABL');

  /* Stable facility key – prefer c_obg/c_obl; fall back to TFC numbers */
  facility_key = cats(c_obg,'_',c_obl);
  if missing(c_obg) and missing(c_obl) then facility_key = coalescec(TFC_Account_Nbr_New, TFC_Account_Nbr);

  /* Keep rows that can be assessed for override */
  if not missing(model_lgd_grade) and not missing(final_lgd_grade);

  /* Housekeeping */
  source_path = vtable; /* table name provenance */
  keep OGM_Qtr segment facility_key model_lgd_grade final_lgd_grade Override_Reason
       TFC_Account_Nbr_New TFC_Account_Nbr c_obg c_obl f_uplddt source_path;
run;

/* ============== 2) FLAG OVERRIDES (exclude operational) ============== */
/* Operational override reasons (from OGM code) to exclude from metric */
proc sql;
  create table OV_CLEAN as
  select *,
         (put(model_lgd_grade,$lgd_num.) ne put(final_lgd_grade,$lgd_num.)) as override_ind,
         /* Exclude operational reasons: 3,4,5,6,7,32,33,36,37,38 */
         (case when strip(Override_Reason) in 
                  ('3','4','5','6','7','32','33','36','37','38')
               then 1 else 0 end)                              as is_operational
  from OV_DISC_RAW
  where not missing(facility_key);
quit;

/* Metric uses only non-operational overrides */
data OV_FOR_RATE;
  set OV_CLEAN;
  if is_operational=1 then override_ind=0;
run;

/* ============== 3) OVERRIDE RATE BY QUARTER × SEGMENT ============== */
proc sql;
  create table OV_RATE as
  select OGM_Qtr,
         segment,
         count(*)                                  as n_facilities,
         sum(override_ind)                         as n_overrides,
         calculated n_overrides / calculated n_facilities format=percent8.2
                                                    as override_rate
  from OV_FOR_RATE
  group by OGM_Qtr, segment
  order by OGM_Qtr, segment;
quit;

/* Optional: overall rate across all segments per quarter */
proc sql;
  create table OV_ALL as
  select OGM_Qtr,
         'ALL' as segment length=3,
         count(*)                  as n_facilities,
         sum(override_ind)         as n_overrides,
         calculated n_overrides / calculated n_facilities format=percent8.2
                                   as override_rate
  from OV_FOR_RATE
  group by OGM_Qtr
  order by OGM_Qtr;
quit;

/* ============== 4) REPEAT-OVERRIDE FACILITIES (cross-quarter) ============== */
proc sql;
  /* Facilities with an override in >=2 different quarters */
  create table REPEAT_FAC as
  select facility_key,
         segment,
         count(distinct OGM_Qtr)          as n_quarters_overridden,
         min(OGM_Qtr)                     as first_qtr,
         max(OGM_Qtr)                     as last_qtr
  from OV_FOR_RATE
  where override_ind=1
  group by facility_key, segment
  having count(distinct OGM_Qtr) >= 2
  order by n_quarters_overridden desc, facility_key;
quit;

/* ============== 5) SANITY SNAPSHOTS (optional to view quickly) ============== */
title "Override Rate by Quarter × Segment";
proc print data=OV_RATE noobs; run;

title "Override Rate by Quarter (All Segments)";
proc print data=OV_ALL noobs; run;

title "Repeat-Override Facilities (>=2 quarters)";
proc print data=REPEAT_FAC(obs=50) noobs; run;
title;

/* ===================== END OF PROGRAM ===================== */
