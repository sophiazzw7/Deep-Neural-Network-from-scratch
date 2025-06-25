import pandas as pd
import numpy as np

def pd_dataset_quality_checks(df: pd.DataFrame):
    # … everything up through your count checks …

    # 7) Date ordering checks – normalize all to tz‑naive before comparing
    date_cols = ["APPROVALDATE", "FUNDEDDATE", "SCORE_DATE", "CHARGEOFFDATE"]
    for d in date_cols:
        # parse everything as UTC then drop the timezone
        ser = pd.to_datetime(df[d], utc=True, errors="coerce")
        ser = ser.dt.tz_localize(None)   # remove the tzinfo
        df[d] = ser

    print("\n7) Date anomalies:")
    print("   Approval > Funded:", (df["APPROVALDATE"] > df["FUNDEDDATE"]).sum())
    print("   Funded > Score_Date:", (df["FUNDEDDATE"] > df["SCORE_DATE"]).sum())
    print("   Chargeoff < Funded:", (df["CHARGEOFFDATE"] < df["FUNDEDDATE"]).sum())

    # … any remaining steps …
