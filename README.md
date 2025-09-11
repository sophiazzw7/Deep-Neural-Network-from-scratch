/* Sort reason counts by quarter, highest mat_cnt first */
proc sort data=abl_by_reason out=abl_reason_sorted;
    by quarter descending mat_cnt reason;
run;

/* Assign ranks within each quarter */
data abl_reason_top10;
    set abl_reason_sorted;
    by quarter;
    retain rank_within_qtr;
    if first.quarter then rank_within_qtr = 0;
    rank_within_qtr + 1;
    if rank_within_qtr <= 10;   /* keep only top 10 for this quarter */
run;

title "Top 10 Material Override Reasons (by frequency within quarter)";
proc print data=abl_reason_top10 noobs label;
    var quarter reason mat_cnt mat_rate;
run;
title;
