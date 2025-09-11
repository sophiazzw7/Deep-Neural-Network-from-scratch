Great catch—and thanks for the screenshot. Those `79-322 / 22-322` errors come from using `ORDER BY` inside a subquery plus `monotonic()` (finicky in PROC SQL). Easiest, bullet-proof fix: **SORT + DATA step ranking** per quarter.

Drop this in (it replaces the earlier ranking bit) ⬇️

```sas
/* Top 10 Material Override Reasons within each quarter */
proc sort data=abl_by_reason out=abl_reason_sorted;
    by quarter descending mat_cnt reason;   /* reason as tiebreaker = deterministic */
run;

data abl_reason_top10;
    set abl_reason_sorted;
    by quarter;
    retain rank_within_qtr;
    if first.quarter then rank_within_qtr = 0;
    rank_within_qtr + 1;
    if rank_within_qtr <= 10;               /* keep only top 10 for this quarter */
run;

title "Material Override Reasons (Top 10 by Material Count within Quarter)";
proc print data=abl_reason_top10 noobs label;
    var quarter reason obs ovr_cnt mat_cnt mat_rate;
run;
title;
```

**Optional (to hide “missing” reasons):**

```sas
data abl_reason_top10;
    set abl_reason_sorted;
    by quarter;
    retain rank_within_qtr;
    if upcase(reason) = 'MISSING' then delete;   /* or exclude earlier in a WHERE */
    if first.quarter then rank_within_qtr = 0;
    rank_within_qtr + 1;
    if rank_within_qtr <= 10;
run;
```

**Quick sanity check (not required, just handy):**

```sas
proc sql;
  select quarter, count(*) as rows
  from abl_reason_top10
  group by quarter;
quit;   /* should be <= 10 for every quarter */
```

This keeps your existing `abl_by_reason` build intact and guarantees **top-10 per quarter** without SQL ranking quirks.
