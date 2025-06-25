import pandas as pd
import numpy as np

# ── assume you’ve already done something like ─────────────────────────────────
# df = pd.read_csv("ZORS_OGM_2024Q2_Data.csv", parse_dates=["APPROVALDATE","FUNDEDDATE","SCORE_DATE","CHARGEOFFDATE"])
# ────────────────────────────────────────────────────────────────────────────────

def pd_dataset_quality_checks(df: pd.DataFrame):
    print("► Shape:", df.shape)
    print("► Duplicate rows:", df.duplicated().sum())
    print("\n1) MISSING VALUES")
    missing = df.isna().sum()
    print(missing[missing > 0].sort_values(ascending=False))
    
    # 2) SCORE sanity: should be in [0,1]
    bad_score = df[(df["SCORE"] < 0) | (df["SCORE"] > 1)]
    print(f"\n2) Out‑of‑bounds SCOREs: {len(bad_score)} rows")
    
    # 3) TARGET distribution (0 = survived, 1 = defaulted)
    print("\n3) TARGET value counts:")
    print(df["TARGET"].value_counts(dropna=False))
    
    # 4) FICO range (300–850 typical)
    bad_fico = df[(df["FICOSCORE"] < 300) | (df["FICOSCORE"] > 850)]
    print(f"\n4) FICO outside [300,850]: {len(bad_fico)} rows")
    
    # 5) PLRS (probability of late payment score) should be ≥ 0
    neg_plrs = df[df["PLRS"] < 0]
    print(f"\n5) Negative PLRS: {len(neg_plrs)} rows")
    
    # 6) Count‑of‑DPD features must be integer & ≥0
    count_cols = [c for c in df.columns if c.startswith("COUNT")]
    for c in count_cols:
        non_int = not pd.api.types.is_integer_dtype(df[c])
        negs = (df[c] < 0).sum()
        print(f"   • {c}: dtype={df[c].dtype}  negative={negs}  non_int={non_int}")
    
    # 7) Date ordering: APPROVAL ≤ FUNDED ≤ SCORE_DATE, CHARGEOFF ≥ FUNDED
    df["APPROVALDATE"] = pd.to_datetime(df["APPROVALDATE"])
    df["FUNDEDDATE"]   = pd.to_datetime(df["FUNDEDDATE"])
    df["SCORE_DATE"]   = pd.to_datetime(df["SCORE_DATE"])
    df["CHARGEOFFDATE"]= pd.to_datetime(df["CHARGEOFFDATE"], errors="coerce")
    
    a_after_f = (df["APPROVALDATE"] > df["FUNDEDDATE"]).sum()
    f_after_s = (df["FUNDEDDATE"] > df["SCORE_DATE"]).sum()
    c_before_f= (df["CHARGEOFFDATE"] < df["FUNDEDDATE"]).sum()
    print(f"\n7) Date anomalies:")
    print(f"   • Approval after Funded: {a_after_f}")
    print(f"   • Funded after Score_Date: {f_after_s}")
    print(f"   • Chargeoff before Funded: {c_before_f}")
    
    # 8) Score calibration by PLRS decile
    df["PLRS_DECILe"] = pd.qcut(df["PLRS"], 10, duplicates="drop")
    summary = df.groupby("PLRS_DECILe").agg(
        avg_TARGET = ("TARGET","mean"),
        avg_SCORE  = ("SCORE","mean"),
        count      = ("TARGET","size")
    )
    print("\n8) Avg default rate vs avg model‐score by PLRS decile:")
    print(summary)
    
# run the checks
pd_dataset_quality_checks(df)
