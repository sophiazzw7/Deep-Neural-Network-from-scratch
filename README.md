Here are the **only fixes that actually matter**:

1. **De-dup rule (the big one)**

```sas
/* WRONG (collapses to one row per obligor) */
by c_obgobl f_uplddt;
if last.c_obgobl;

/* RIGHT (OGM spec: keep last per obligor+upload date) */
by c_obgobl f_uplddt;
if last.f_uplddt;
```

2. **Segment flag (small but easy to get wrong)**

```sas
/* Use case-insensitive equality */
SF = (upcase("&seg.") = 'SF');
```

3. **Operational-reason filtering (type-safe)**
   Make sure `Override_Reason` is **character** when excluding:

```sas
if put(input(Override_Reason, ?? best12.), 12.) in ('3','4','5','6','7','32','33','36','37','38') then delete;
/* or if it’s already char codes: */
if Override_Reason in ('3','4','5','6','7','32','33','36','37','38') then delete;
```

Everything else in your pipeline is fine. The incorrect de-dup (`last.c_obgobl`) is what was suppressing events/repeats.
