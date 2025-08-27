/* ==== 0) Inputs: point to the two snapshots ==== */
%let DS_PRIOR = /sasdata/.../all_table_2024Q4.sas7bdat;   /* t-1 */
%let DS_CUR   = /sasdata/.../all_table_2025Q1.sas7bdat;   /* t   */

/* ==== 1) Prep each snapshot: pick keys + fields, compute utilization ==== */
proc format; value $lg('A'=1,'B'=2,'C'=3,'D'=4,'E'=5,'F'=6,'G'=7,'H'=8,'I'=9,'J'=10,'K'=11,other=.); run;

%macro prep(in=, out=);
data &out;
  set &in(keep=TFC_Account_Nbr_New c_obg c_obl LOB TFC_rsk_grd tfc_face_amt tfc_curr_bal
                /* keep collateralclass only if it’s already present in your all_table */
                collateralclass );
  /* Keep ABL & SF only; require a positive commitment */
  where (LOB in ('ABL','SF')) and tfc_face_amt>0;

  /* Facility key (use post-merger key when available) */
  length acct_key $60;
  acct_key = coalescec(TFC_Account_Nbr_New, cats(c_obg,'|',c_obl));

  /* Weighted utilization */
  util = tfc_curr_bal / tfc_face_amt;

  /* Orderable risk-grade rank: number + letter */
  band  = input(compress(TFC_rsk_grd,,'ka'), 8.);     /* numeric part, e.g., 3 in 3A */
  letter= compress(TFC_rsk_grd, '0123456789 ');
  notch = input(put(letter,$lg.), 8.);                 /* A=1 … K=11 */
  grd_rank = band*100 + notch;                         /* sortable rank */
  keep acct_key LOB TFC_rsk_grd grd_rank util tfc_curr_bal tfc_face_amt collateralclass;
run;
%mend;

%prep(in=&DS_PRIOR, out=prior);
%prep(in=&DS_CUR  , out=cur  );

/* ==== 2) Pair the same facilities present in BOTH periods ==== */
proc sql;
create table paired as
select  a.acct_key, a.LOB,
        a.TFC_rsk_grd as grade_t1, a.grd_rank as rank_t1,
        a.util as util_t1, a.tfc_curr_bal as draw_t1, a.tfc_face_amt as com_t1,
        b.TFC_rsk_grd as grade_t0, b.grd_rank as rank_t0,
        b.util as util_t0, b.tfc_curr_bal as draw_t0, b.tfc_face_amt as com_t0,
        coalesce(a.collateralclass,b.collateralclass) as collateralclass
from cur a inner join prior b
  on a.acct_key = b.acct_key;
quit;

/* Direction and magnitude of rating change */
data paired;
  set paired;
  d_rank = rank_t1 - rank_t0;  /* >0 = downgrade, <0 = upgrade */
  length mig $10;
  if d_rank>0 then mig='Downgrade';
  else if d_rank<0 then mig='Upgrade';
  else mig='No change';

  /* bins for size of change (optional) */
  abschg = abs(d_rank);
  length size $6;
  if abschg=0 then size='0';
  else if 1<=abschg<=200 then size='1-2';  /* one or two notches */
  else if 201<=abschg<=400 then size='3-4';
  else size='5+';
run;

/* ==== 3) Kevin’s table: utilization before vs after, by migration ==== */
proc sql;
create table util_by_mig as
select LOB, mig,
       sum(draw_t0)/sum(com_t0) as util_prev format=percent8.1,
       sum(draw_t1)/sum(com_t1) as util_curr format=percent8.1,
       calculated util_curr - calculated util_prev as delta format=percent8.1,
       count(*) as n_facilities
from paired
group by LOB, mig
order by LOB, calculated delta desc;
quit;

/* Optional: add collateral class split (ABL most relevant; 4=AR,5=AR-Govt,6=Inv) */
proc sql;
create table util_by_mig_collat as
select LOB, collateralclass, mig,
       sum(draw_t0)/sum(com_t0) as util_prev format=percent8.1,
       sum(draw_t1)/sum(com_t1) as util_curr format=percent8.1,
       calculated util_curr - calculated util_prev as delta format=percent8.1,
       count(*) as n_facilities
from paired
group by LOB, collateralclass, mig;
quit;
