**Subject:** KS bootstrap vs. back-test mismatch

Hi \[Name],

A bootstrap run on the same six build samples, using the standard two-sample KS (goods vs bads), should average out to the plain KS means in the MDD (\~0.64 Auto, 0.45 DC, etc.). The appendix shows 0.02–0.06 instead, so something’s off.

Could you check:

1. Metric — are we using one-sample KS instead of two-sample?
2. Data — is the bootstrap limited to funded/seasoned loans or missing the train slice?

Once those line up, the means should match and we can set meaningful thresholds.

Thanks,
Phoebe
