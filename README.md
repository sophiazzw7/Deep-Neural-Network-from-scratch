/* Combine into one row */
data sim_bands;
    merge b_Q50(rename=(B50_P2=P2_Q50 B50_P50=P50_Q50 B50_P97=P97_Q50))
          b_Q75(rename=(B75_P2=P2_Q75 B75_P50=P50_Q75 B75_P97=P97_Q75))
          b_Q90(rename=(B90_P2=P2_Q90 B90_P50=P50_Q90 B90_P97=P97_Q90))
          b_Q95(rename=(B95_P2=P2_Q95 B95_P50=P50_Q95 B95_P97=P97_Q95))
          b_Q99(rename=(B99_P2=P2_Q99 B99_P50=P50_Q99 B99_P97=P97_Q99));
run;
