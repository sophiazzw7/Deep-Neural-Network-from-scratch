/* ---------- build _ABL from quarterly OVERRIDE_*_DEDUP_ABL ---------- */

/* empty shell (so PROC APPEND has a base) */
data _ABL;
  length quarter $7 qtr_idx 8
         c_obgobl $200 f_uplddt 8
         override_ind 8 OVRD_SEVERITY 8
         model_lgd_grade_num 8 final_lgd_grade_num 8
         material_1notch 8
         /* ids we need later */
         STI_obl_nbr 8 c_obl 8 fpb_lgdappid 8
         Override_Reason $200 TFC_Curr_Bal 8 tfc_face_amt 8;
  stop;
run;

/* small macro to keep the code tiny */
%macro add_qtr(q=, ds=);
data _ABL_&q.;
  length quarter $7;
  set &ds (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
                 model_lgd_grade_num final_lgd_grade_num
                 STI_obl_nbr c_obl fpb_lgdappid
                 Override_Reason TFC_Curr_Bal tfc_face_amt);
  quarter="&q.";
  qtr_idx = input(substr(quarter,1,4),8.)*10 + input(substr(quarter,6,1),8.);

  /* backfill severity if needed (your existing rule) */
  if missing(OVRD_SEVERITY) and nmiss(final_lgd_grade_num,model_lgd_grade_num)=0 then
    OVRD_SEVERITY = final_lgd_grade_num - model_lgd_grade_num;

  if missing(override_ind) then override_ind = (OVRD_SEVERITY ne 0);
  material_1notch = (abs(OVRD_SEVERITY) > 1);
run;

proc append base=_ABL data=_ABL_&q. force; run;
proc datasets lib=work nolist; delete _ABL_&q.; quit;
%mend;

/* call for the four quarters you’re using */
%add_qtr(q=2024Q2, ds=ogm.OVERRIDE_2024Q2_DEDUP_ABL);
%add_qtr(q=2024Q3, ds=ogm.OVERRIDE_2024Q3_DEDUP_ABL);
%add_qtr(q=2024Q4, ds=ogm.OVERRIDE_2024Q4_DEDUP_ABL);
%add_qtr(q=2025Q1, ds=ogm.OVERRIDE_2025Q1_DEDUP_ABL);

/* ---------- obligation + LGD approval repeat (add-on, leaves your obligor logic intact) ---------- */


