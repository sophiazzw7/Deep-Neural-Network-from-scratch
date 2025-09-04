/*=============================*
 | 0) LIBRARIES (edit paths)   |
 *=============================*/
options mprint mlogic symbolgen nosource2;
libname q2 "/sasdata/mrmx/MOD1497/2024Q2_v2.1";
libname q3 "/sasdata/mrmx/MOD1497/2024Q3_v2.1";
libname q4 "/sasdata/mrmx/MOD1497/2024Q4_v2.1";
libname q1 "/sasdata/mrmx/MOD1497/2025Q1_v2.2";

/*=============================*
 | 1) STACK THE QUARTERS       |
 *=============================*/
/* Assumptions (present in join_1497_final_res):
   - model_lgd_grade, final_lgd_grade
   - Override_Reason (character code)
   - TFC_Account_Nbr_New (or TFC_Account_Nbr)
   - c_obg, c_obl (for facility key), f_uplddt
   - SF (1=SF, 0/blank=ABL) or STI_lob_nm usable to infer segment
*/
data OV_DISC_RAW;
  length OGM_Qtr $6 segment $3;
  set 
    q2.join_1497_final_res(in=a)
    q3.join_1497_final_res(in=b)
    q4.join_1497_final_res(in=c)
    q1.join_1497_final_res(in=d);

  if a then OGM_Qtr='2024Q2';
  else if b then OGM_Qtr='2024Q3';
  else if c then OGM_Qtr='2024Q4';
  else if d then OGM_Qtr='2025Q1';

  /* Segment */
  if not missing(SF) then segment = ifc(SF=1,'SF','ABL');
  else segment = 'ABL'; /* fallback if SF not provided */

  /* Stable facility key: prefer c_obg||c_obl; else account */
  length facility_key $60;
  facility_key = cats(c_obg, '_', c_obl);
  if missing(c_obg) and missing(c_obl) then facility_key = coalescec(TFC_Account_Nbr_New, TFC_Account_Nbr);

  /* Keep only rated rows that matter (both grades populated) */
  if not missing(model_lgd_grade) and not missing(final_lgd_grade);

  format f_uplddt datetime20.;
run;

/*=============================*
 | 2) FLAG OVERRIDES           |
 *=============================*/

/* Map letter grades to numbers (A=1 … T=20) to compute severity */
proc format;
  value $lgd_num
    'A'=1  'B'=2  'C'=3  'D'=4  'E'=5  'F'=6  'G'=7  'H'=8  'I'=9  'J'=10
    'K'=11 'L'=12 'M'=13 'N'=14 'O'=15 'P'=16 'Q'=17 'R'=18 'S'=19 'T'=20;
run;

/* Exclude operational overrides (same list used in OGM code) */
%let OP_EXCL = 3 4 5 6 7 32 33 36 37 38;

data OV_DISC_CLEAN;
  set OV_DISC_RAW;

  /* standardize reason field to simple code (numeric or char both okay) */
  length _reason $2;
  _reason = compress(put(input(compress(Override_Reason,' '), best.), 2.));
  /* If Override_Reason is already character codes like '3','33', this keeps them */

  /* Flag override (non-operational) */
  override_ind = (strip(upcase(final_lgd_grade)) ne strip(upcase(model_lgd_grade)));

  /* drop operational reasons from “material override” set */
  if override_ind then do;
    if _reason in: ('3','4','5','6','7','32','33','36','37','38') then override_ind=0;
  end;

  /* Severity (positive means upgraded toward better/lower LGD if final<model) */
  model_num = input(put(upcase(model_lgd_grade), $lgd_num.), best.);
  final_num = input(put(upcase(final_lgd_grade), $lgd_num.), best.);
  if nmiss(model_num, final_num)=0 then ovrd_severity = final_num - model_num; else ovrd_severity = .;

  keep OGM_Qtr segment facility_key model_lgd_grade final_lgd_grade override_ind ovrd_severity
       TFC_Account_Nbr_New TFC_Account_Nbr c_obg c_obl f_uplddt;
run;

/*=============================*
 | 3) QUARTER x SEG OVERRIDES  |
 *=============================*/
proc sql;
  create table OV_RATE as
  select OGM_Qtr,
         segment,
         count(*)                             as n_facilities,
         sum(override_ind)                    as n_overrides,
         calculated n_overrides / calculated n_facilities format=percent8.2 as override_rate
  from OV_DISC_CLEAN
  group by OGM_Qtr, segment
  order by OGM_Qtr, segment;
quit;

/*=============================*
 | 4) REPEAT-OFFENDER CHECK    |
 *  (facility overridden in 2+ quarters)
 *=============================*/
proc sql;
  /* 4a. Which facilities ever overridden? */
  create table OV_ONLY as
  select distinct OGM_Qtr, segment, facility_key
  from OV_DISC_CLEAN
  where override_ind=1;

  /* 4b. Count quarters per facility */
  create table REPEAT_FAC as
  select segment,
         facility_key,
         count(distinct OGM_Qtr) as n_quarters_overridden
  from OV_ONLY
  group by segment, facility_key
  having calculated n_quarters_overridden >= 2
  order by segment, n_quarters_overridden desc;
quit;

/*=============================*
 | 5) DIAGNOSTICS / COMPARISON |
 *=============================*/
title "Override Rate by Quarter & Segment (Material Overrides Only)";
proc print data=OV_RATE noobs; run;

title "Facilities Overridden in 2+ Quarters";
proc print data=REPEAT_FAC(obs=50) noobs; run;
title;

/* Optional: per-quarter override severity distribution */
proc means data=OV_DISC_CLEAN n mean median p25 p75;
  class OGM_Qtr segment;
  var ovrd_severity;
  where override_ind=1;
run;

/* Optional: export for Excel spot checks */
*filename xout "/sasdata/mrmx/MOD1497/ovr_summary_v22.xlsx";
*proc export data=OV_RATE    outfile=xout dbms=xlsx replace; sheet="OV_RATE";    run;
*proc export data=REPEAT_FAC outfile=xout dbms=xlsx replace; sheet="REPEAT_FAC"; run;
*proc export data=OV_DISC_CLEAN outfile=xout dbms=xlsx replace; sheet="ROW_DETAIL"; run;
*filename xout clear;
