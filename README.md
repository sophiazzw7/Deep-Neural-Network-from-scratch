/* sort by gross_loss */
proc sort data=severity_et2_10k( where=(gross_loss >= &L_trunc and gross_loss ne .) )
          out=severity_et2_10k_sorted;
    by gross_loss;
run;
