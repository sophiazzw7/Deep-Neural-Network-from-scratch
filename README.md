
1. Size of ABL portfolio (from your table)
* 2024Q2: ABL 256 obligations, $3,139,504,212 outstanding, $13,522,654,620 commitments. SF 57, $117,151,970, $752,366,366.
* 2024Q3: ABL 239, $2,874,773,493, $13,805,358,272. SF 75, $133,669,674, $1,150,863,438.
* 2024Q4: ABL 230, $3,066,910,019, $12,804,800,199. SF 78, $157,092,392, $1,229,845,002.
* 2025Q1: ABL 219, $3,277,447,829, $11,565,463,725. SF 82, $182,557,785, $1,365,681,439. Latest quarter total ≈ $3.46B outstanding across ~301 obligations; commitments ≈ $12.93B.
2. Model use (exactly as the MDD states)
* Purpose: generate LGD ratings at obligation level; supports the Dual-Risk Rating with PRISM PD.
* Not forecasting: “NOT implemented in any loss-forecasting processes or systems of models.” (i.e., not CCAR/CECL).
* Downstream examples (indirect): Commercial Loan Pricing and Profitability Management may use PRISM LGD as one input for expected loss/profitability.
* Output form: LGD letter grade mapped to a % on the PRISM LGD Masterscale and staged in LAS.
3. Typical collateral & EAD treatment for ABL lines (from the sheets + MDD text)
* Collateral classes present (ABL): Accounts Receivable, Accounts Receivable–Government, Inventory, Others, Unsecured. • In one “Collateral Class – ABL” tab (counts 535→460), percents shown are roughly AR ~39%, Inventory ~33%, Others ~23%, Unsecured ~5%, AR-Govt ~1%. • A different tab you shared (smaller counts) shows a higher AR share (~mid-60s%). Since both are your files, I’d cite the distribution from the “Collateral Class – ABL” tab with the larger counts unless you want the other cut—happy to reflect whichever tab you plan to present.
* Control/monitoring mix (ABL): Monitored & Perfected Security Interest ≈97%, Perfected SI ≈2%, Other ≈1%.
* EAD for ABL facilities (this is the precise scorecard logic): • Revolver EAD = max { 95% × (Total Group Borrowing Base − Blocks − Reserves), (Total revolver outstanding + Total Pari-Passu outstanding) }, capped by total commitments (Truist + pari-passu). • Term/LC EAD: 100% Term/Finance LC; 50% Performance LC; 25% Trade LC. • Blocks = fixed dollar withheld; Reserves = dynamic dollar withheld to cover liquidation/other costs. (Correction to my earlier shorthand: it is not simply “drawn + accruals”; it follows the borrowing-base formula above.)
4. How the EOS/scorecard defines LGD (from the MDD formulas and tables)
* Structural identity: LGD = 1 − RR / EAD. • RR (recoveries) = ROS + ROU − Reserves. • ROS (secured) uses Recoverable Collateral Value (RCV) parameters that differ by collateral type and valuation method (NFVL/NOLV/FMV). • ROU (unsecured) assumes 5% recovery on the unsecured slice; 3rd-party guarantees provide a lift (use the maximum lift among guarantee types; the sheet shows the lift and an additional +5% rule when multiple guarantees are present). • The MDD shows the rearrangement used to back into RCV parameters from target LGDs and advance-rate guidance supplied by the ABL group. • Example guidance table (from the MDD): Inventory (NFVL) LGD 8% / advance rate 95%; Equipment (NFVL) LGD 15% / 90%; AR 5% / 85%; AR-Govt 5% / 90%.
* Raw LGD formula displayed: Raw LGD = 1 − (ROU + ROS − Reserves) / Total Group EAD, then apply any caps/floors and map to the LGD Masterscale (A=0–4%, B=4–7.5%, C=7.5–12.5%, …).
5. High-level LGD observation (what your distribution shows)
* Your LGD Risk Rating Distribution – Total chart shows A and B dominate (A ≈ high-30s% final, B ≈ low-30s% final in the screenshot), with thinner mass in C, D, E, etc.
* Using the masterscale bands shown (A 0–4%, B 4–7.5%, C 7.5–12.5%, …), that profile supports describing the book as low-LGD under the bank’s scale.
Quick explainers (plain language, taken from the screenshots’ context)
* Borrowing Base: the eligible value of collateral you lend against; the scorecard uses 95% × (Borrowing Base − Blocks − Reserves) in the EAD max test.
* Blocks: fixed dollars the bank won’t lend against.
* Reserves: dynamic dollars withheld to cover liquidation/other costs.
* Pari Passu: other lenders at the same priority; their outstanding is added in the EAD max test.
* NFVL / NOLV / FMV: collateral valuation methods referenced in the RCV tables.
* Masterscale: lookup table that converts the computed LGD into a letter grade (A, B, C, …) used in LAS and Dual-Risk Rating.
If you’d like, I can paste this into a one-pager and (optionally) add two exhibits straight from your numbers: (i) the quarter-by-quarter size table and (ii) the LGD grade histogram + collateral/control bars.
