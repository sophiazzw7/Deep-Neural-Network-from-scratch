Here’s the same list without bold formatting:

---

### Data Quality and Preparation

1. Could you share the code and details used to check data reasonbleness, missing values, and duplicates?
2. The Critical Data Elements (CDEs) are listed in the MDD, but the rationale for their selection is not documented. Can you confirm whether a rationale exists and where it’s maintained?
3. Are variable units and formats for the model inputs documented anywhere? They are not shown in the current MDD.
4. The MDD does not include basic descriptive statistics (mean, min, max, standard deviation) for the development data. Could you provide or point to where these are stored?
5. In the qualitative development model code, where are outliers identified and excluded? 



### Model Development and Implementation Consistency

6. For the severity and frequency datasets in the output folder, can you confirm that these are the exact datasets used for frequency fitting? 
7. Does the development code match the implementation code currently in production? If not, please identify what components are missing or modified.
8. MRO notes that documentation could be strengthened by including both qualitative evidence (e.g., parameter instability or convergence issues for ET4) and quantitative metrics (AIC, KS, Q–Q fit) to demonstrate why LDA was deemed unsuitable below the 1,000-event threshold. Can these details be added to the MDD?



### Documentation and Implementation Traceability

9. Please include the implementation link or reference path in the MDD to support traceability.
10. The Excel file used to manually fit the baseline from the distribution should be included and accompanied by an explanation of the process.
11. The model pipeline description in the MDD needs to be cleaned up—currently it only contains qualitative model code. Could you update it to reflect the full end-to-end process, including data preparation, fitting, and simulation steps?
