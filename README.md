import pandas as pd
import numpy as np

def pd_dataset_quality_checks(df: pd.DataFrame):
    # 0) drop the nested‑dict column(s) that break hashing
    qc = df.drop(columns=["DATA"], errors="ignore")
    
    print("► Shape:", qc.shape)
    print("► Duplicate rows (ignoring DATA):", qc.duplicated().sum())

    print("\n1) MISSING VALUES")
    missing = qc.isna().sum()
    print(missing[missing > 0].sort_values(ascending=False))

    # 2) SCORE sanity: should be in [0,1]
    bad_score = qc[(qc["SCORE"] < 0) | (qc["SCORE"] > 1)]
    print(f"\n2) Out‑of‑bounds SCOREs: {len(bad_score)} rows")

    # 3) TARGET distribution
    print("\n3) TARGET value counts:")
    print(qc["TARGET"].value_counts(dropna=False))

    # 4) FICO range (300–850 typical)
    bad_fico = qc[(qc["FICOSCORE"] < 300) | (qc["FICOSCORE"] > 850)]
    print(f"\n4) FICO outside [300,850]: {len(bad_fico)} rows")

    # 5) PLRS must be ≥0
    neg_plrs = qc[qc["PLRS"] < 0]
    print(f"\n5) Negative PLRS: {len(neg_plrs)} rows")

    # 6) Count‑of‑DPD columns
    count_cols = [c for c in qc.columns if c.startswith("COUNT")]
    for c in count_cols:
        negs = (qc[c] < 0).sum()
        is_int = pd.api.types.is_integer_dtype(qc[c])
        print(f"   • {c}: dtype={qc[c].dtype}, negatives={negs}, integer={is_int}")

    # 7) Date ordering checks
    for d in ["APPROVALDATE","FUNDEDDATE","SCORE_DATE","CHARGEOFFDATE"]:
        qc[d] = pd.to_datetime(qc[d], errors="coerce")
    print("\n7) Date anomalies:")
    print("   Approval > Funded:", (qc["APPROVALDATE"] > qc["FUNDEDDATE"]).sum())
    print("   Funded > Score_Date:", (qc["FUNDEDDATE"] > qc["SCORE_DATE"]).sum())
    print("   Chargeoff < Funded:", (qc["CHARGEOFFDATE"] < qc["FUNDEDDATE"]).sum())

    # 8) Calibration by PLRS decile
    qc["PLRS_DECILE"] = pd.qcut(qc["PLRS"], 10, duplicates="drop")
    calib = qc.groupby("PLRS_DECILE").agg(
        avg_default=("TARGET","mean"),
        avg_score=("SCORE","mean"),
        n=("TARGET","size")
    )
    print("\n8) Calibration check (by PLRS decile):")
    print(calib)

# usage
pd_dataset_quality_checks(df)
