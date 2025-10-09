Short answer: **don’t rely on `unique_id` for dupes until you prove it’s 1–1 with the business key the ETL uses.**
In this pipeline, the *business* dedupe/join key is:

**`(company_id, start_date, lob_indicator, sample_tp)`**

Use that for duplicate checks. `unique_id` may be:

* built differently in different stages (e.g., `company_id|rpt_prd_date_id` in flat),
* missing `lob_indicator` / `sample_tp` (so one `unique_id` could map to multiple cube rows), or
* a surrogate/row key (always unique → useless for dupes).

Below is a tiny SAS block to **prove** whether `unique_id` is safe to use in the cube. If it’s not strictly 1–1 with the business key, don’t use it for dupes.

---

### 1) Validate how `unique_id` behaves in the cube

```sas
/* Build the canonical business key string used by the ETL merges */
data _cube_keys;
  set cubes.bdfs_am_final_summary_202412;
  length bizkey $120;
  bizkey = cats(strip(company_id),'|',put(start_date,yymmddn8.),'|',strip(lob_indicator),'|',strip(sample_tp));
run;

/* A) Does one unique_id point to multiple business keys? (bad for duping) */
proc sql;
  create table _uid_to_many_biz as
  select unique_id, count(distinct bizkey) as n_bizkeys
  from _cube_keys
  group by unique_id
  having calculated n_bizkeys > 1;
quit;

/* B) Does one business key have multiple unique_id? (also bad) */
proc sql;
  create table _biz_to_many_uid as
  select bizkey, count(distinct unique_id) as n_uids
  from _cube_keys
  group by bizkey
  having calculated n_uids > 1;
quit;

title "unique_id → multiple business keys (should be 0 rows if unique_id is safe)";
proc print data=_uid_to_many_biz (obs=20); run;

title "business key → multiple unique_id (should be 0 rows if unique_id is safe)";
proc print data=_biz_to_many_uid (obs=20); run;
```

**Interpretation**

* If both tables come back **empty**, `unique_id` is 1–1 with the business key → you *may* use `unique_id` for dup checks.
* If **any rows appear**, `unique_id` is not 1–1 → **use the business key** for dupes.

---

### 2) Duplicate checks (recommended way = business key)

```sas
/* Canonical duplicate check in the cube on ETL business key */
proc sort data=cubes.bdfs_am_final_summary_202412
          out=_cube_dedup
          dupout=_cube_dups
          nodupkey;
  by company_id start_date lob_indicator sample_tp;
run;

title "Duplicates on ETL business key (company_id, start_date, lob_indicator, sample_tp)";
proc sql; select count(*) as dup_rows from _cube_dups; quit;

proc print data=_cube_dups (obs=20);
  var company_id start_date lob_indicator sample_tp score_value weight;
  format start_date date9.;
run;
```

*(If and only if the test above proves `unique_id` is 1–1, you could alternatively do `by unique_id;`—but the business key remains the ground truth.)*

---

### 3) For `f_app_score_flat`

* In the **flat**, developers typically set `unique_id = company_id || '|' || rpt_prd_date_id`.
* For your sanity check slice, we rebuild **`start_date = EOM(score_date)`** and use the **natural key `(company_id, start_date)`** to check dupes (since flat might lack the exact cube `unique_id` or `rpt_prd_date_id`):

```sas
/* Rebuild start_date and check dupes in the flat (CURRENT, <= eop12) */
%let rpt_prd_date_id = 20241231;
%let eop12n = %sysfunc(intnx(month,%sysfunc(inputn(&rpt_prd_date_id,yymmdd8.)),-12,end));

data _flat_expected;
  set model.f_app_score_flat;
  length start_date 8; format start_date date9.;
  start_date = coalesce(intnx('month', score_date, 0, 'end'), eff_from_dt);
  if sample_tp='CURRENT' and start_date <= &eop12n;
run;

proc sort data=_flat_expected out=_fe_dedup dupout=_fe_dups nodupkey;
  by company_id start_date;
run;

title "Duplicates in f_app_score_flat on (company_id, start_date)";
proc sql; select count(*) as dup_rows from _fe_dups; quit;

proc print data=_fe_dups (obs=20);
  var company_id start_date score_date eff_from_dt score_value weight;
  format start_date date9.;
run;
```

---

## Bottom line

* **Use the ETL business key** `(company_id, start_date, lob_indicator, sample_tp)` for the cube.
* Only use `unique_id` for deduping **if** you verify it’s a strict 1–1 with that business key (the quick test above tells you).
