Subject: KS Bootstrap Mean vs. Back-Testing Mean – results don’t reconcile

Hi \[Developer’s Name],

I took another look at the appendix.  If the bootstrap routine is run on the same six data partitions (test\_1, test\_2, valid\_1, valid\_2, train, hold-out) **and** uses the same two-sample KS definition (score distribution of goods vs. bads), its average KS should converge to the plain “mean KS” reported in Section 6.1.1 – that’s a property of bootstrapping.  In other words:

```
E[KS_bootstrap]  ≈  KS_original_sample
```

In the MDD those original-sample means are \~0.64 (Auto), 0.45 (Debt Consolidation), 0.55 (Home Improvement), etc., yet the appendix shows bootstrap means in the 0.02–0.06 range.  A gap that large suggests we’re no longer looking at the same statistic or the same population.

Could you check:

1. **KS definition** – are we using a one-sample KS (probability vs. binary target) instead of the standard two-sample KS?
2. **Data scope** – are the train slice or any high-risk rejects excluded from the bootstrap?
3. **Other filters** – funded-only or 18-month-seasoning requirements that wouldn’t apply to the build partitions?

Once the bootstrap is aligned with the back-testing dataset and metric, its mean should validate the \~0.45–0.64 KS numbers and give us a sound basis for setting thresholds.

Let me know what you find or if I can help run a quick reproducibility check.

Thanks!

Best,
Phoebe
