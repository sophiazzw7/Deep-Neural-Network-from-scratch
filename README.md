/* In EACH of your per-quarter steps (ABL_q2, ABL_q3, ABL_q4, ABL_q1):           */
/* just ADD these names to the KEEP= list on the SET statement.                  */
/* Example shown for q3 — repeat the same edit in q2/q4/25Q1 blocks.             */

data _ABL_q3;
  length quarter $7 seg $3 c_obgobl $200 obligor_key $200 material_1notch 8;
  set ogm.OVERRIDE_2024Q3_DEDUP_ABL
      (keep=c_obgobl f_uplddt override_ind OVRD_SEVERITY
            model_lgd_grade_num final_lgd_grade_num
            /* === ADD THESE THREE === */
            STI_obl_nbr c_obl fpb_lgdappid
      );
  ...
run;
