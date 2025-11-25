Got it—this is really clear now.

### 1️⃣ Math check

From the screenshots:

* **Frequency ±20% table (IMG_3321)**
  Using
  [
  \text{pct shift} = \frac{\text{new} - \text{original}}{\text{original}}
  ]
  I re-calculated for all ET2/ET3/ET7 intercepts and alphas. Your numbers (e.g.

  * ET2 Intercept: +20% → **3.21% / 2.99%**, –20% → **–3.91% / –3.66%**
  * ET3 Intercept: +20% → **5.61% / 5.30%**, –20% → **–6.83% / –6.50%**
  * ET7 Intercept: +20% → **4.42% / 4.29%**, –20% → **–5.41% / –5.22%**

  all match the calculations to two decimal places. ✅

* **Severity ±5% table (IMG_3322)**
  I re-did the same for all TE/LN intercepts and LN scales. Your large moves (e.g. ET3 LN scale –5% ≈ **+44.5% / +31.1%**, ET7 TE intercept –5% ≈ **+47.5% / +48.2%**, ET7 LN scale –5% ≈ **–56.1% / –57.5%**) are all correct to rounding. ✅

* The **tail-increase CIs** you show in the last two screenshots are also consistent with the original CIs and with your pass/fail patterns.

So you’re safe to use these four charts; the arithmetic is right.

---

### 2️⃣ Polite reply draft to Cheng (mentioning the four charts)

You can send something like this (assuming you attach or paste the four charts):

---

**Email draft**

Hi Cheng,

Thank you for your detailed questions and for taking the time to look closely at our comments. I prepared a few additional sensitivity summaries to clarify what we meant by “too insensitive” on the frequency side and “too volatile” on the severity side, as well as our comment on the CI–overlap threshold.

**1. Frequency parameter sensitivity**

In the attached frequency charts (Charts 1 and 3), we looked at how the parameter CIs change under ±20% data shocks and under 10–20% tail-increase scenarios:

* A **±20% change in the underlying data** produces only **single-digit percentage shifts** in the frequency parameter CIs (for example, intercept and alpha CIs generally move by about **3–9%**).
* Even with **20% tail increases**, the CI-overlap test continues to classify the parameters as “stable” for all base event types.

This is why we described the frequency metric as **too insensitive for use as an early-warning indicator**: relatively large changes in the underlying frequency or tail behavior are still treated as “no issue” by the CI-overlap rule.

**2. Severity parameter volatility**

In the attached severity charts (Charts 2 and 4), we see the opposite pattern:

* A relatively small **±5% change in severity** leads to **very large movements in some severity parameters**, with certain intercepts and scale parameters shifting by **30–50%** in their CIs.
* For ET3, even with a **10–20% increase in the tail**, the CI-overlap test still passes, despite these sizeable parameter shifts.

This is why we referred to the severity side as **overly volatile** and the current CI-overlap cutoff as **too loose** for detecting meaningful tail drift.

**3. On “sensitivity” and “volatility” thresholds**

Our comments were based on these model-specific observations rather than on a universal numeric rule. MRO does not maintain a single percentage threshold that defines “sensitive” or “volatile” across all models. Instead, we assess whether:

* the stability metric reacts proportionally to **material changes in the underlying risk profile**, and
* the level of variation is reasonable for use as an **effective early-warning indicator**.

For this particular model, the combination of small parameter movement on the frequency side under large shocks, and very large parameter movement on the severity side under small shocks, led us to conclude that the current CI-overlap implementation is not well balanced for OGM.

If you find it helpful, we are happy to walk through these charts together and discuss options for re-designing the stability and tail-drift metrics so they better support early-warning monitoring.

Thank you,
Phoebe

---

If you’d like, I can also shorten this or make a version that refers to each chart by exact title/figure number depending on how you paste them into your email.
